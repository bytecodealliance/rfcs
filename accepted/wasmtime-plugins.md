# Summary
[summary]: #summary

Enable `wasmtime` plugin functionality via component composition and allowing components to load dynamic libraries using [wasi-dl]

# Motivation
[motivation]: #motivation

On a high level, WebAssembly components more often than not require certain capabilities from the host at runtime to fulfill their tasks, for example, access to network or file system are common examples of such capabilities.
As WebAssembly adoption grows, so does the variety of capabilities that are required by WebAssembly components and applications.
Various WASI proposals are developed to address this need, which are then implemented in `wasmtime` itself and custom embeddings of it.

Currently `wasmtime` comes bundled with a limited set of WASI functionality and optionally enabled proposals such as `wasi-http`, `wasi-nn` etc.

Providing custom host interface implementations for either bundled interfaces or completely custom ones requires a custom `wasmtime` embedding.

The status quo results in a few issues:

- Every additional WASI proposal bundled in `wasmtime` increases the maintenance burden
- Many WASI proposals (like `wasi:keyvalue` or `wasi:nn`) abstract over details of concrete implementations, however they require integrating with those concrete implementations on the host side to be useful.
For example, `wasi:keyvalue` host interface implementations are most useful when they are able to interact with real key-value stores.
Apart from additional maintenance burden, adding support for these concrete implementations pollutes the dependency graph of `wasmtime` itself, which in turn:
    - increases the binary size
    - makes `wasmtime` more susceptible to supply-chain attacks
    - negatively affects build speeds
- Integrations with some services may not be possible to be implemented in `wasmtime` or even as part of a Bytecode Alliance project due to licensing incompatibilities.

From perspective of `wasmtime` embedders, host interface implementations are tightly coupled with `wasmtime` version and so are a significant maintenance burden, especially if maintained outside of `wasmtime` tree.
`wasmtime` linker API as well as `wasmtime` WASI implementation are effectively the API surface that developers must target to extend `wasmtime` embeddings.
Maintainers of, for example, Rust crates that provide host interface implementations require a release cadence synchronised with `wasmtime` with each release being a breaking change due to `wasmtime` crate version increasing the major version.

Even though WebAssembly components become more and more capable over time, the need for direct access to host functionality (e.g. access to hardware devices) is unlikely to become redundant any time soon.

Plugin use case requirements vary greatly - for example, while for some users running plugins as part of their WebAssembly runtime process may be acceptable, for others it may not be.

See https://github.com/bytecodealliance/wasmtime/issues/7348 for additional context on the CLI use case

# Proposal
[proposal]: #proposal

A WebAssembly runtime provides capabilities to WebAssembly components by means of implementing WIT interfaces in terms of capabilities provided to the runtime itself by the operating system.
For example, `wasi:filesystem` is implemented in terms of disk I/O capabilities provided (or not) to the `wasmtime` process by the operating system.

A host runtime plugin also needs to implement some set of WIT interfaces to provide a set of capabilities to the component, however there is one key difference between the two - whereas a WebAssembly runtime is a process running on an operating system, a plugin does not need to be one.

In this proposal, I would like to suggest using WebAssembly (reactor) components as host runtime plugins.

Since plugins are WebAssembly components, there is no need for any plugin-specific functionality to be built into `wasmtime` - instead, WebAssembly components can be *composed* with the plugin component ahead-of-time to produce a single component, which is then executed in `wasmtime`, just like a regular component. Composability of WebAssembly components is then also used to compose multiple plugins into a single one.

`wasmtime` already provides sufficient access to operating system capabities, like networking or filesystem access required to implement a big part of potential host plugins. For example, on a high level, implementation of `wasi:keyvalue` in terms of a connection to one's database of choice by using `wasi:sockets` either directly or via a standard library integration is already *possible*, albeit often difficult.

The biggest blocker for majority of such plugins being pure WebAssembly components is broad ecosystem adoption - keeping `wasi:keyvalue` as the example, many databases provide integration libraries in various languages, however usage of these libraries in WebAssembly components is often problematic or simply impossible out-of-the-box. Lack of features normally available in native applications, like threading, may also be problematic.

In order to address the lack of WebAssembly adoption in third-party libraries, lacking WebAssembly features and allow integration with existing projects, I suggest yet another WASI proposal to be built into `wasmtime`: [wasi-dl].

## [wasi-dl]

[wasi-dl] is a WASI proposal, which ultimately abstracts over [`dlfcn.h`](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/dlfcn.h.html) allowing components to link to arbitrary dynamic libraries at runtime and access symbols provided by them.

### Interface

Core functionality, extracted from https://github.com/rvolosatovs/wasi-dl/blob/59136630adc3b4c5cd69714d3446180d790cab73/wit/dl.wit looks as follows:

```wit
package wasi:dl@0.2.0-draft;

interface ffi {
    resource alloc {
        new: static func(types: list<ffi-type>) -> result<alloc>;
    }

    // FFI type definitions etc.
}

interface dll {
    use ffi.{alloc, ffi-type, outgoing-value, incoming-value};

    resource function {
        /// Constructs a function from an opaque `alloc` and a type signature
        constructor(alloc: alloc, args: list<ffi-type>, ret: option<ffi-type>);

        call: func(args: list<outgoing-value>) -> result<option<incoming-value>>;
    }

    flags open-flags {
        lazy,
        now,
        global,
        local,
    }

    resource library {
        open: static func(name: option<string>, flag: open-flags) -> result<library, string>;

        get: func(name: string) -> result<alloc, string>;
    }

    extension: func() -> string;
    prefix: func() -> string;
    suffix: func() -> string;
}
```

Such interface is *unsafe* and it must be used with extreme care, however that is no different from any other host plugin, which would be loaded via `dlopen`. 

[wasi-dl] attempts to provide reasonably safe abstractions to components for the unsafe operations abstracted over.

### Usage

Example usage in a Rust component, which invokes `char *hello(void)` provided by a library loaded using a relative path:

```rust
let suf = dll::suffix();
let pre = dll::prefix();
let lib = dll::Library::open(
    Some(&format!("./path/to/{pre}hello{suf}")),
    dll::OpenFlags::NOW,
)
.expect("failed to open library");

let sym = lib.get("hello").expect("failed to get `hello` symbol");
let func = sym.function(vec![], Some(FfiType::Primitive(PrimitiveType::Pointer)));

let res = func
    .call(vec![])
    .expect("failed to call `hello`")
    .expect("`hello` is missing a return value");

let IncomingValue::Primitive(res) = res else {
    panic!("`hello` return value is not a primitive");
};
let res = res
    .get_alloc()
    .expect("failed to get `hello` return value pointer")
    .read_string()
    .expect("failed to read `hello` string");
assert_eq!(res, "hello");
```

### Host implementation

Since the function signature, from the perspective of `wasmtime` is only known at runtime when it is communicated to the host by the component, I propose using [libffi] for calling the dynamically-typed loaded functions.

[libffi] is a well-established and popular cross-platform project used by many langugage toolchains (e.g. `CPython`, `OpenJDK` etc.)

This allows components to integrate with majority of existing software libraries out-of-the-box.

### PoC

For a PoC of [wasi-dl] implemented via [libffi] in `wasmtime` see https://github.com/rvolosatovs/wasmtime/commit/c7f23e4133db93deb16779f597b2447648372365

## RPC-based plugins

Naturally, RPC-based functionality can be implemented in a dynamic library loaded via [wasi-dl], however that also be done using "portable" WebAssembly component plugins, not relying on native libraries.

For example, a WebAssembly component, which exports a single `hello` function, which, using `wasi:sockets`, is proxied over TCP to a [wRPC] service looks like the following:

```rust
mod bindings {
    use crate::Handler;

    wit_bindgen::generate!({
        world: "server",
        with: {
            "wrpc-examples:hello/handler": generate,
        },
    });
    export!(Handler);
}

mod wrpc_bindings {
    wit_bindgen_wrpc::generate!({
        world: "client",
        with: {
            "wrpc-examples:hello/handler": generate
        },
    });
}

struct Handler;

async fn invoke(addr: &str) -> String {
    let wrpc = wrpc_transport::tcp::Client::from(addr);
    let (pollables_tx, mut pollables_rx) = tokio::sync::mpsc::unbounded_channel();
    let (s, _) = tokio::join!(
        async {
            wrpc_bindings::wrpc_examples::hello::handler::hello(&wrpc, pollables_tx)
                .await
                .expect("failed to invoke `wrpc-examples.hello/handler.hello`")
        },
        // Drive asynchronous I/O using pollables, this will be redundant in WASI 0.3
        drive_io(pollables_rx),
    );
    s
}

impl bindings::exports::wrpc_examples::hello::handler::Guest for Handler {
    fn hello() -> String {
        tokio::runtime::Builder::new_current_thread()
            .build()
            .expect("failed to create runtime")
            .block_on(invoke("[::1]:7761"))
    }
}
```

Full example can be found at https://github.com/bytecodealliance/wrpc/blob/e3c961325b3dad58e51cb405d24aadaaae5146e4/examples/rust/hello-component-tcp-proxy/src/lib.rs

Example run instructions available in https://github.com/bytecodealliance/wrpc/pull/403#issue-2597167135

## Conclusion

With this proposal, we have 3 different ways of providing WIT interface implementations to components:

- Built-in into the runtime
- Portable WebAssembly component plugin, not relying on [wasi-dl]
- A shared library, which implements functionality, which is used by a WebAssembly component plugin via [wasi-dl] and exported

So how do we choose the correct approach for a particular interface or use case?

I suggest adopting the following set of guidelines:

1. `wasmtime` repository should be hardware and software-vendor agnostic, however `wasmtime` may use specific features of a platform it's running on (e.g. features available on a particular operating system, but not on others).

2. `wasmtime` interface implementations must not target specific services, but may rely on widely used standards.

3. `wasmtime` should strive to provide built-in implementations for only a limited subset of core, low-level, WASI proposals, which largely mimic capabilities normally provided by the host operating system.

   `wasi:sockets`, `wasi:cli` or `wasi:clocks` are good examples of such proposals.

4. `wasmtime` may provide built-in implementations of higher-level WASI proposals, for example: `wasi:blobstore` or `wasi:keyvalue`.

   `wasmtime` may provide integration crates that embedders can use to target a concrete implementation in their own custom embeddings.
   For example, for `wasi:keyvalue`, `wasmtime-wasi-keyvalue` crate should:
   - define a reusable Rust store trait
   - allow embedders to customize `open` function implementation
   - allow embedders to choose an appropriate implementation of the Rust store trait, most likely, based on the value given to `open`
   
   Concrete implementations of abstractions provided by `wasmtime` integration crates (like `wasmtime-wasi-keyvalue`) targeting specific services (like a database) should be maintained outside the `wasmtime` repository.
    
5. WASI proposal repositories should provide hardware and software-vendor agnostic WebAssembly component implementations (plugins).

   WASI proposal repositories may additionally provide WebAssembly component and `wasmtime` crate integration implementations targeting specific hardware and software vendors or their products, as long as their licenses are compatible.

   WebAssembly component plugins not relying on [wasi-dl], if possible, should always be the preferred option.


   For example, `wasi:keyvalue` repository should provide a reference WebAssembly component implementation using in-memory storage as a plugin.
   There should be a CI job, which composes the reference component with a test component *importing* `wasi:keyvalue` and runs the resulting component in `wasmtime`.

   `wasi:keyvalue` repository may also provide WebAssembly components and `wasmtime-wasi-keyvalue` implementations targeting specific databases and key-value stores.
   Such component plugin implementations may rely on [wasi-dl] to load external (e.g. provided by the vendor) or custom shared libraries to enable the integration.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## [wasi-dl]

As an alternative to exposing [wasi-dl] interface to (plugin) components, we could use the dynamic libraries themselves as the host plugins.
For that we would need to carefully design a set of conventions specific to `wasmtime` for such plugins to be able to define their exports and expose them to components.

Such an approach would require custom-built dynamic libraries for plugins, if an existing library was desired to be used, an "adapter" library would need to be built, which would in turn dynamically-load that library.

With [wasi-dl], components can load *any* existing shared library and reuse the already-existing tooling to define their exports (like `wit-bindgen`).

[wasi-dl] also provides greater portability and performance for components - whereas with shared-libraries-as-plugins approach components run by `wasmtime` would import a set of interfaces provided by these plugins, with [wasi-dl] such plugins would likely only depend on a few core interfaces and [wasi-dl] itself.

[wasi-dl]-based approach also provides greater security, since the implementation of [wasi-dl] may restrict the set of libraries allowed to be loaded and potentially define the exact signatures for symbols defined in them.

## [libffi]

Without a custom convention or having C header files and processing them, signature of an arbitrary function loaded from a shared object cannot be known.
Instead of using [libffi], wasmtime could choose to handle this issue directly, which would require platform-specific implementations and likely assembly code.
While such an approach is possible and perhaps feasible to pursue in the future, it requires a lot of work and dependency on [libffi] seems to be a small price to pay, especially since it is very common on modern systems.

[libffi] does not have support for all C23 types, but it seems to have enough to cover overwhelming majority of use cases. See more documentation on the topic at: https://www.chiark.greenend.org.uk/doc/libffi-dev/html/Types.html

## Separate plugin interface

Whether or not the plugins are WebAssembly components, `wasmtime` could expose an API to configure them as a unit separate to the component being run/served.
Such an approach raises a few design questions without a simple, clear answer, for example:
- How do plugin imports work together with host builtins? Should `wasmtime` throw an error if e.g. both built-in `wasi:keyvalue` *and* `wasi:keyvalue` plugin import are specified?
- It is not clear whether machinery allowing plugins to interact with host resources is required or even a desired feature at all. For example, could a plugin export a function, which would take `wasi:http/types.fields` resource as a parameter, where `wasi:http/types.fields` is exported by the host? If so, the host would need to export `wasi:http/types.fields` methods to allow the plugin to interact with it.
- How to compose multiple plugins?

By relying on component composition these questions disappear and the user gains full control of this behavior, which can be adjusted to a particular use case at hand, while `wasmtime` can focus on executing a single compnent it in a secure sandbox.

# Open questions
[open-questions]: #open-questions

- Should a set of guidelines or rules be defined for the (new, custom) loadable shared objects?

[libffi]: https://sourceware.org/libffi
[wasi-dl]: https://github.com/rvolosatovs/wasi-dl
[wRPC]: https://github.com/bytecodealliance/wrpc
