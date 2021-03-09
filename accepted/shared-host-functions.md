# Summary

[summary]: #summary

This RFC proposes adding support in the Wasmtime API for sharing host function definitions between multiple stores.

# Motivation
[motivation]: #motivation

Wasmtime is currently well-suited for long-lived module instantiations, where a singular `Instance` is created for a 
WebAssembly program and the program is run to completion.
 
In this model, the time spent defining host functions for the instance is negligible because the setup is only performed once 
and the instance is, potentially, expected to run for a long time.
 
However, this model is not ideal for services that create short-lived instantiations for each service request, where the time 
spent creating host functions to import at instantiation-time can be considerable for the repeated creation of many instances.
 
As an instance is expected to live only for the duration of the request, each instance would be associated with its own `Store` 
that is dropped when the request completes.
 
Given the nature of `Store` isolation in Wasmtime, this means host function definitions must be recreated for each and every 
request handled by the service.
 
If instead host functions could be defined for a `Config` (in addition to `Store`), then the repeated effort to define the 
host functions would be eliminated and the time it takes to create an instance to handle a request would therefore be 
noticeably reduced.

# Proposal
[proposal]: #proposal

As `Func` is inherently tied to a `Store`, it cannot be used to represent a function associated with an `Config`.
 
This RFC proposes not having an analogous type to represent these functions; instead, users will define the function via 
methods on `Config`, but use `Store` (or a `Linker` associated with a `Store`) to get the `Func` representing the function.
 
## Overview example
 
A simple example that demonstrates defining a host function in a `Config`:
 
```rust
let mut config = Config::default();

config.wrap_host_func("", "hello", |caller: Caller, ptr: i32, len: i32| {
    let mem = caller.get_export("memory").unwrap().into_memory().unwrap();
    println!(
        "Hello {}!",
        std::str::from_utf8(unsafe{ &mem.data_unchecked()[ptr..ptr + len] }).unwrap()
    );
});

let engine = Engine::new(&config);
let module = Module::new(
    &engine,
    r#"
    (module
        (import "" "hello" (func $hello (param i32 i32)))
        (memory (export "memory") 1)
        (func (export "run")
            i32.const 0
            i32.const 5
            call $hello
        )
        (data (i32.const 0) "world")
    )
    "#,
)?;
 
let store = Store::new(module.engine());
let linker = Linker::new(&store);
 
let instance = linker.instantiate(&module)?;
let run = instance
    .get_export("run")
    .unwrap()
    .into_func()
    .unwrap()
    .get0::<()>();
 
run()?;
```
 
## Changes to `wasmtime::Config`
 
This RFC proposes that the following methods be added to `Config`:
 
```rust
/// Defines a host function for the [`Config`] for the given callback.
///
/// Use [`Store::get_host_func`] to get a [`Func`] representing the function.
///
/// Note that the implementation of `func` must adhere to the `ty`
/// signature given, error or traps may occur if it does not respect the
/// `ty` signature.
///
/// Additionally note that this is quite a dynamic function since signatures
/// are not statically known. For performance reasons, it's recommended
/// to use [`Config::wrap_host_func`] if you can because with statically known
/// signatures the engine can optimize the implementation much more.
///
/// The callback must be `Send` and `Sync` as it is shared between all engines created
/// from the `Config`.  For more relaxed bounds, use [`Func::new`] to define the function.
pub fn define_host_func(
    &mut self,
    module: &str,
    name: &str,
    ty: FuncType,
    func: impl Fn(Caller<'_>, &[Val], &mut [Val]) -> Result<(), Trap> + Send + Sync + 'static,
);

/// Defines a host function for the [`Config`] from the given Rust closure.
///
/// Use [`Store::get_host_func`] to get a [`Func`] representing the function.
///
/// See [`Func::wrap`] for information about accepted parameter and result types for the closure.
///
/// The closure must be `Send` and `Sync` as it is shared between all engines created
/// from the `Config`.  For more relaxed bounds, use [`Func::wrap`] to wrap the closure.
pub fn wrap_host_func<Params, Results>(
    &mut self,
    module: &str,
    name: &str,
    func: impl IntoFunc<Params, Results> + Send + Sync,
);
```

Similar to `Func::new`, `define_host_func` will define an host function that can be used for any Wasm function type.

Similar to `Func::wrap`, `wrap_host_func` will generically accept different `Fn` signatures to determine the WebAssembly type of 
the function.
 
These methods will internally create an `InstanceHandle` to represent the host function.
 
However, the instance handle will not owned by a store (as is the case with `Func`); instead, the `Config` will own the associated instance handles and deallocate them when dropped.

Note: the `IntoFunc` trait is documented as internal to Wasmtime and will need to be extended to implement this feature;
therefore that will not be considered a breaking change.
 
## Changes to `wasmtime::Store`
 
To use host functions defined on a `Config` as imports for module instantiation or as `funcref` values, a `Func` representation is 
required.

This proposal calls for adding the following methods to `Store`:
 
```rust
/// Gets a host function from the [`crate::Config`] associated with this [`Store`].
///
/// Returns `None` if the given host function is not defined.
pub fn get_host_func(&self, module: &str, name: &str) -> Option<Func>;

/// Gets a context value from the store.
///
/// Returns a reference to the context value if present.
pub fn get<T: Any>(&self) -> Option<&T>;

/// Sets a context value into the store.
///
/// Returns the given value as an error if an existing value is already set.
pub fn set<T: Any>(&self, value: T) -> Result<(), T>;
```
 
`get_host_func` will register the function's instance handle with the `Store`, but as a *borrowed* handle which it is 
not responsible for deallocation. As `Store` keeps a reference on its associated `Engine` (and therefore `Config`), this ownership model should be 
sufficient to prevent `funcref` values or imports of shared host functions from outliving the function instance itself.

For host functions that require contextual data (e.g. WASI), the `set` method will be used for associating context with 
a `Store` and it can later be retrieved via `caller.store().get()`.
 
## Changes to `wasmtime::Linker`
 
There will be no changes to the API surface of `Linker` to support the changes proposed in this RFC.
 
However, the `Linker` implementation will be changed such that it will fallback to calling `Store::get_host_func` when 
resolving imports for `Linker::instantiate` and `Linker::module`.
 
This should allow for overriding shared host function imports in the `Linker` if an import of the same name has already been defined 
in the associated `Config`.
 
## Changes to `wasmtime_wasi::Wasi`
 
`Wasi` is a type generated from `wasmtime-wiggle`.
 
This proposal adds the following methods to the generated `Wasi` type:
 
```rust
/// Adds the WASI host functions to the given [`Wasmtime::Config`].
///
/// Host functions added to the config expect [`Wasi::set_context`] to be called.
///
/// WASI host functions will trap if the context is not set in the calling [`wasmtime::Store`].
pub fn add_to_config(config: &mut Config);

/// Sets the context in the given store.
///
/// This method must be called on a store that imports a WASI host function when using `Wasi::add_to_config`.
///
/// Returns the given context as an error if a context has already been set in the store.
pub fn set_context(store: &Store, ctx: WasiCtx) -> Result<(), WasiCtx>;
```
 
`add_to_config` will add the various WASI functions to the given `Config`.

The function implementations will expect that `set_context` has been called for the `Store` prior to any invocations.

If the context is not set, the WASI function implementations will trap.

### WASI shared host function example

```rust
let mut config = Config::default();
Wasi::add_to_config(&config);

let engine = Engine::new(&config);
 
let module = Module::new(
    &engine,
    r#"
    (module
        (import "wasi_snapshot_preview1" "fd_write" (func $fd_write (param i32 i32 i32 i32) (result i32)))
        (memory (export "memory") 1)
        (func (export "run")
            i32.const 1
            i32.const 0
            i32.const 1
            i32.const 12
            call $fd_write
            drop
        )
        (data (i32.const 0) "\0C\00\00\00\0D\00\00\00\00\00\00\00Hello world!\n")
    )
    "#,
)?;

let store = Store::new(module.engine());

// Set the WasiCtx in the store
// Without this, the call to `fd_write` will trap
assert!(Wasi::set_context(&store, WasiCtxBuilder::new().build()?).is_ok());

let linker = Linker::new(&store);
let instance = linker.instantiate(&module)?;

let run = instance
   .get_export("run")
   .unwrap()
   .into_func()
   .unwrap()
   .get0::<()>();
 
run()?;
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

No other designs have been considered.

# Open questions
[open-questions]: #open-questions

* Should this be exposed via Wasmtime-specific C API functions?

* Is `Store` the right place for storing context?
  For WASI, this means all instances that use the same store will share the `WasiCtx`.
