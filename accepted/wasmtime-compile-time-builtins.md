# Summary
[summary]: #summary

Add support for defining builtin host functions at compile-time. Because these
functions are early-bound at compile-time -- rather than late-bound at
instantiation-time, like regular imports -- the Wasm's compilation can be
specialized for these exact imports, enabling inlining without just-in-time
compilation, for example. This is the rough equivalent of the
[js-string-builtins proposal][js-string-builtins] but for Wasmtime's API and
Wasmtime embedding environments rather than JavaScript's `WebAssembly` API and
JavaScript execution environments.

[js-string-builtins]: https://github.com/WebAssembly/js-string-builtins/blob/main/proposals/js-string-builtins/Overview.md

# Motivation and Requirements
[motivation]: #motivation

We have two primary, and related, desiderata:

1. **Remove call overheads for simple host functions.** For simple getter- and
   setter-style functions, call overhead can dwarf the execution time of the
   function body itself, particularly when we consider that the function
   boundary hides information from Cranelift, inhibiting optimizations like GVN
   and LICM between the caller and callee. Furthermore, with today's late-bound
   imports, and without a tiering JIT compiler or mutating executable code,
   direct calls to an imported function at the Wasm level must become indirect
   calls through a [PLT/GOT]-like mechanism at the machine code level, imposing
   additional overhead. We would like to eliminate these costs.

2. **Enable performant, zero-copy manipulation of host buffers exposed to Wasm
   as component model resources.** A host system often already has data in
   external buffers, outside of a Wasm instance, and wants to grant Wasm access
   to that data. This requires choosing between copying the data into the Wasm's
   linear memory or giving the Wasm a file descriptor-style handle to an object
   that represents that external buffer and its contents are read and written
   via additional function imports. This latter approach successfully avoids the
   cost of copying the data into Wasm linear memory, but introduces function
   call overheads to all accesses. However, if the read and write accessor
   functions are simple enough, we can inline them with this RFC's proposal,
   eliminating the function call overheads while still avoiding copies.

[PLT/GOT]: https://reverseengineering.stackexchange.com/a/1993

Note that our goal is improving performance in the scenario where Wasm is
importing host functions, and not the scenario where one Wasm module is linking
to another Wasm module, satisfying its own imports with the other's
exports. Host functions are compiled at a different moment in time and by a
different compiler from Wasm compiled by Wasmtime. Host functions can access
memory and resources outside of any Wasm instance's sandbox. These properties
tie our hands and constrain our potential solutions.

Additionally, note that the definition of compile-time builtins is pretty
fundamentally `unsafe`: you are interacting with the guts of Wasm code
generation and promising things like "I pinky swear that accessing this host
memory at this pointer is equivalent to what my native host function would have
otherwise done". Similarly, because these compile-time builtins must have the
ability to load from and store to native memory, they cannot be portable across
ISAs with different pointer widths.

It is worth noting that our goals here are not new or unique; there is plenty of
prior art. Even beyond the js-string-builtins proposal, Web browsers have been
inlining simple DOM methods that are normally implemented in host C++ code, like
`Element.prototype.id` for example, into JS code for a very long time.

Finally, we also have one hard requirement:

1. **We must not deviate from the WebAssembly language semantics.** Whether an
   import is early-bound at compile-time or late-bound at instantiation-time
   must be invisible to the Wasm program itself. All else being equal, should a
   component virtualize an interface rather than use a builtin host
   implementation, for example, that decision must not be semantically visible
   to the interface's consumer. The performance characteristics may change; the
   semantics must not. Wasmtime remains committed to open standards.

# Potential Approaches
[alternatives]: #alternatives

There are three viable alternatives for defining compile-time builtins that I
have identified:

1. Expose CLIF in Wasmtime's public API

2. Define a new mini-language

3. Self-host Wasm, giving it access to privileged intrinsics

Each of these can satisfy our requirements. In that sense, any one of them would
be acceptable. However, there are three additional dimensions along which it is
important that we evaluate them:

1. **API Burden:** How much additional API surface area are we committing to
   maintaining? How hard will it be to keep these APIs relatively stable?

2. **Implementation Burden:** How much new code will we need to implement? How
   hard will it be to maintain?

More details on each approach will follow, but for the purpose of quickly
reviewing their tradeoffs, I've summarized our three alternatives along these
additional dimensions in the following table:

| Candidate Solution | API Burden | Implementation Burden |
|---|---|---|
| Expose CLIF | ⭐★★★★ | ⭐⭐⭐⭐⭐ |
| Mini-Language | ⭐⭐⭐⭐⭐ |  ⭐⭐⭐★★ |
| Self-hosted Wasm | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

## 1. Expose CLIF in Wasmtime's Public API

One option is to simply define API hooks that expose a
`cranelift_frontend::FunctionBuilder` whenever a intended-for-inlining host
function is called from Wasm code, and then let embedders use those APIs to emit
the body of the corresponding host function inline directly.

```rust
let mut code_builder = wasmtime::CodeBuilder::new();

// ...

code_builder.define_compile_time_builtin(
    "foo",
    "bar",
    wasmtime::FuncType::new([ValType::I32, ValType::I32], [ValType::I32]),
    |builder: &mut cranelift_frontend::FunctionBuilder, args: &[cranelift_codegen::ir::Value]| {
        // Use `builder` to emit the compile-time builtin inline. Can use the
        // usual builder methods to load from and store to native memory.
        let [x, y] = args else { unreachable!() };
        builder.ins().iadd(x, y)
    },
);
```

Because this exposes all of our CLIF-building APIs to users directly, none of
which have any sort of formal stability, this rates very poorly from an API
burden perspective.

As far as implementation burden goes, however, this is very straightforward. We
already have the types and methods, we just re-export them, and expose hooks for
getting instances of them at the correct times.

Finally, it does not seem possible to support Winch with this approach. That is
not ideal but ultimately deemed okay, since if you really care about runtime
speed, you should be using Cranelift anyways.

## 2. Define a New Mini-Language

Similar to the mini-language used for defining `cranelift_codegen::ir::Global`s,
we could define a mini-language for defining compile-time builtins. This would
have various arithmetic and memory operations, but would not be a fully general,
Turing-complete language.

```rust
let mut code_builder = wasmtime::CodeBuilder::new();

// ...

// Use our mini-language's builder APIs to define a compile-time builtin function.
let mut builtin_builder = code_builder.define_compile_time_builtin(
    "foo",
    "bar",
    wasmtime::FuncType::new([ValType::I32, ValType::I32], [ValType::I32]),
);
let [x, y] = builtin_builder.args() else { unreachable!() };
let z = builtin_builder.add32(x, y);
builtin_builder.return_(z);
```

The API burden is minimal: we can define exactly what operations we want to
support and not add anything else into the mini-language.

The implementation burden is slightly higher, but not too bad, since we need to
actually implement the mini-language and translate it into CLIF or Winch API
calls.

> Note: I have a working, proof-of-concept prototype of this approach in [my
> `compile-time-builtins-mini-language`
> branch](https://github.com/bytecodealliance/wasmtime/compare/main...fitzgen:wasmtime:compile-time-builtins-mini-language). For
> example, [this
> test](https://github.com/bytecodealliance/wasmtime/blob/17a0780675adf108d9683558577230b6a9cbd4f0/tests/all/compile_time_builtins.rs#L28-L32)
> defines a compile-time builtin, compiles a core module that imports and calls
> said builtin, checks that the import is inlined and erased from the resulting
> `Module`, calls the module's exported function, asserts that the result is as
> expected, and this test is passing.

## 3. Use Self-Hosted Wasm with Privileged Intrinsics

The final alternative is to allow compile-time builtins to be defined as
self-hosted Wasm functions that have access to special intrinsics (imported
functions under the, say, `__wasmtime_intrinsics` namespace that is only exposed
to compile-time builtins) for reading and writing native memory. Rather than
compiling these self-hosted functions down into their own native functions, we
would instead translate them to CLIF or Winch inline whenever we see calls to
them.

```rust
let mut code_builder = wasmtime::CodeBuilder::new();

// ...

code_builder.define_compile_time_component_instance(
    "foo",
    r#"
        (component
            ;; ...
        )
    "#,
)?;
```

The API burden is minimal: WebAssembly is already precisely defined and we can
take Wasm binaries as input rather than provide a builder with a bunch of
methods.

The maintenance burden is also minimal: we must already parse and generate code
for Wasm.

# Proposal

As for how to define compile-time builtins, this RFC proposes that we move
forward with the Wasm self-hosting approach.

First, we add a `wasmtime::CodeBuilder::define_compile_time_component_instance`
method:

```rust
impl CodeBuilder {
    pub unsafe fn define_compile_time_component_instance(
        &mut self,
        name: &str,
        self_hosted_wasm: &[u8],
    ) -> Result<&mut Self> {
        // ...
    }
}
```

This is the compile-time equivalent of
[`wasmtime::component::Linker::instance`](https://docs.rs/wasmtime/latest/wasmtime/component/struct.Linker.html#method.instance).

`self_hosted_wasm` is a Wasm component binary (or WAT if the `"wat"` cargo
feature is enabled). We validate it and propagate any errors.

If the user component that the code builder is compiling imports an instance named
`name`, then we will do the following:

* Remove the import of `name`
* Make it so that the user component effectively contains an inline `(component
  ...)` definition of the compile-time component
* Extend the user component's imports with the compile-time component's imports
* Make it so that the user component's execution begins by instantiating the
  compile-time component, forwarding imports to the compile-time component
* Use the compile-time component's instance wherever the imported `name`
  instance was used

Additionally, the compile-time component will be allow-listed access to Wasmtime
intrinsics and may import these functions. Intrinsic imports will not be
forwarded to the user component, unlike other imports, and will instead have
their associated operation directly inlined during compilation. Giving access to
these intrinsics is also why the method is `unsafe`: callers are promising that
the self-hosted Wasm is well-behaved, trusted code and that it will not misuse
the intrinsics to access invalid memory, cause data races, touch Wasmtime's
internal data structures, or etc...

Finally, note that when [Wasmtime's support for function
inlining](https://github.com/bytecodealliance/wasmtime/pull/11283) is enabled,
the compile-time component's functions (and any associated adapters we generate)
can be inlined into callers, removing all function call overheads.

## Restrictions on Self-Hosted Wasm

There are no restrictions on the shape of self-hosted Wasm components. They may
define multiple core Wasm modules that in turn define multiple memories,
globals, and tables. Their functions (including imported intrinsic functions)
may be `ref.func`ed. Anything that normal Wasm can do, self-hosted Wasm can also
do.

As an incremental milestone, however, the "MVP" implementation of self-hosted
Wasm might not support all these operations in their entirety.

## Intrinsics

This is a list of the intrinsics that will be available for compile-time
components as an instance with the name `__wasmtime_intrinsics`. Calls to
intrinsics do not become actual function calls, they are replaced with a handful
of native instructions.

Most intrinsics are various load and store operations for the native address
space. The pointer type is always a `u64`, but its high 32 bits are ignored on
32-bit systems. This allows writing compile-time builtins that are portable
across builds targeting different ISAs. If we required the use of `u32` on
32-bit ISAs, then compile-time builtin authors would need to maintain both a 32-
and 64-bit variant of their builtins. The `native` mnemonic hints that the
memory operation is operating on the native memory address space, not a
particular Wasm memory, and uses native endianness.

The final intrinsic, `resource.address`, gives the address of the host data in
the resource table for a particular resource handle. It will raise a trap on
invalid resources and out-of-bounds resource table accesses.

```wat
(import "__wasmtime_intrinsics"
  (instance
    (export "u8.native_load" (func (param "address" u64) (result u8)))
    (export "u16.native_load" (func (param "address" u64) (result u16)))
    (export "u32.native_load" (func (param "address" u64) (result u32)))
    (export "u64.native_load" (func (param "address" u64) (result u64)))

    (export "i8.native_load" (func (param "address" u64) (result i8)))
    (export "i16.native_load" (func (param "address" u64) (result i16)))
    (export "i32.native_load" (func (param "address" u64) (result i32)))
    (export "i64.native_load" (func (param "address" u64) (result i64)))

    (export "u8.native_store" (func (param "address" u64) (param "value" u8)))
    (export "u16.native_store" (func (param "address" u64) (param "value" u16)))
    (export "u32.native_store" (func (param "address" u64) (param "value" u32)))
    (export "u64.native_store" (func (param "address" u64) (param "value" u64)))

    (export "i8.native_store" (func (param "address" u64) (param "value" i8)))
    (export "i16.native_store" (func (param "address" u64) (param "value" i16)))
    (export "i32.native_store" (func (param "address" u64) (param "value" i32)))
    (export "i64.native_store" (func (param "address" u64) (param "value" i64)))

    ;; Note: this signature is not actually valid, see open questions.
    (export "resource.address" (func (param "resource" (borrow (sub resource))) (result u64)))
  )
)
```

Note that we do *not* define intrinsics for directly accessing or addressing the
`vmctx`, any linear memories, or any other internal state of the Wasm
instance. While Wasmtime needs those abilities to implement operations like
`global.get` and `memory.size`, compile-time builtins do not get to see inside
Wasmtime's implementation details. They are only given helpers for accessing
data that the embedder themselves defined, such as the host object backing a
Wasm resource.[^blah]

[^blah]: Of course, although they should only ever access embedder-defined data
    as part of their safety contract, compile-time builtins ultimately have the
    capability to access absolutely anything -- including Wasmtime internals --
    since they have native address space loads and stores. That doesn't mean we
    need to add more footguns than we fundamentally must.

While we needn't implement all of these intrinsics from the very start, we
should in the fullness of time implement all of them. In the process of
implementing compile-time builtins, we may also recognize oversights in the
above list and add additional intrinsics or change existing ones.

## Compile-time core modules?

The above proposal only defines a mechanism for defining compile-time Wasm
*components* not *core modules*. There is no reason we cannot add a
`wasmtime::CodeBuilder::define_compile_time_module_instance` method for
compile-time core module instance. It is simply excluded from this RFC because
our current use cases all use component interfaces. Once the above proposal for
compile-time components is implemented, it should be relatively straightforward
to reuse that infrastructure for core modules, and we can discuss that
possibility at that future point in time.

## Winch and compile-time builtins?

There was originally some question around whether it even made sense to allow
compile-time builtins with Winch, since Winch will not inline calls to these
compile-time builtins, defeating most of their motivation. However, satisfying a
Wasm's imports at compile time, making it so that they need not be provided at
instantiation time, is still valuable for Winch users. Furthermore, nothing laid
out in the above proposal is fundamentally incompatible with Winch, or relies on
anything specific to any of our Wasm compilers. The only compiler-specific piece
is the implementation of each of the Wasmtime intrinsics' stubs, and these are
isolated and should be relatively straightforward to implement. Therefore, while
we might skip Winch support in an initial "MVP" implementation, there is no
reason not to support compile-time builtins with Winch in the fullness of time.

# Open questions
[open-questions]: #open-questions

* **How do we specify which resource type's table we want to access in the
  `resource.address` intrinsic?**

  The `resource.address` signature above is not actually valid: the component
  model's interface types do not provide a way to define a function that takes
  *any* resource, regardless of where or when it was defined, as an
  argument. And in fact, if I remember correctly, we use different index spaces
  for different types of resources, so resource index `r` could be valid in
  multiple different resource tables.

  This intrinsic kind of wants to be a canonical builtin (like `resource.drop`
  or `future.new`) rather than a regular function, so we can provide a resource
  type as an immediate to disambiguate between different types of resources. But
  defining new canonical builtins is entering the realm of extending the Wasm
  language, rather than just providing powerful imports to certain components,
  and I don't think we should go down that route.

  I suppose we could provide the type index of the resource type as a dynamic
  argument -- in practice it should always be a constant so we can figure out
  which resource table to access at compile time. But if it is not a constant,
  what do we even do? Call out to the host? The whole point of this feature is
  to avoid such calls... Perhaps this isn't so bad, and it just becomes another
  array indirection: index into the array of resource tables, then index into
  the array of table elements?

  Anyone have any other ideas?
