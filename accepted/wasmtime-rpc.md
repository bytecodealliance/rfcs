# Summary
[summary]: #summary

Allow extending `wasmtime` CLI with RPC-based host plugins.

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

The proposal is to allow extending `wasmtime` CLI with out-of-tree, RPC-based host plugins and use [wRPC] as the (first, potentially out of many) RPC protocol.

This would be an *addition* to existing functionality, rather than a replacement of it, which would compose with what is already available.

The CLI `run` and `serve` commands would support an additional option, it seems that `-P` for `--plugin` would fit well:

```
  -P, --plugin <KEY[=VAL[,..]]>
          Plugin-related configuration options, `-P help` to see all
```

```
Available plugin options:

  -P protocol=<protocol> --

        Selects the plugin protocol to use, e.g. `wrpc+tcp` or `wrpc+unix`
  -P target=<target> --

        Specifies the protocol-specific plugin target, which provides
        imports.
        For `wrpc+tcp`, this is a resolvable address, e.g. `localhost:7761`
        For `wrpc+unix`, this is a path on the file system e.g. `/tmp/wasmtime-rpc`
  -P import=<spec>[,spec[,..]] --

        Selects the interfaces to import using the plugin.
        The value is a sequence comma-separated interface specifiers.
        For example: `wasi:keyvalue` or `wasi:keyvalue,wasi:nn`
```

Example usage:

```
wasmtime run -P protocol=wrpc+tcp -P target=[::1]:7761 -P import=wasi:keyvalue,wasi:nn my.wasm
```

To improve usability, Wasmtime could standardize on default socket address or Unix domain socket path to use.
For example, I chose port `7761` as the default for TCP, which is specified as "unassigned" in [IANA registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?&page=108).

<!-- 7761 is obtained by joining hex encoding of characters `w` and `a`, i.e. `format!("{:02x}", (u32::from('w') << 8) | u32::from('a'))` in Rust. -->

With a default of `protocol=wrpc+tcp` and `target=[::1]:7761`, the invocation could look like:

```
wasmtime run -P import=wasi:keyvalue,wasi:nn my.wasm
```

In spirit of this being a composable, additive change, I would suggest that the same `-P` option namespace would be reused for the FFI plugin model when/if it becomes available.
For FFI use case, perhaps `protocol=ffi` or something similar could be used to select it.

During instantiation of the component, Wasmtime would iterate over all component imports and for each import matching a specification passed to `import` flag, it would define a new function in the linker, which would invoke the import over chosen wRPC transport (specified in `target`).
An example implementation of a [wRPC] "polyfill" can be found in the [wRPC Wasmtime integration crate](https://github.com/bytecodealliance/wrpc/blob/153cfc1bdbb487c4a413061a8f64d706107e47d1/crates/runtime-wasmtime/src/lib.rs#L1004-L1108).

In case that the plugin is not reachable, the import would simply trap.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

[wRPC] is chosen as the RPC protocol for this RFC. See [(draft) specification](https://github.com/bytecodealliance/wrpc/blob/main/SPEC.md) for lower-level details.

## Light on dependencies

[`wrpc-transport`](https://docs.rs/wrpc-transport/latest/wrpc_transport/) crate, which would be the integration point for Wasmtime is built on top of well-known Rust crates, which are already in Wasmtime dependency graph:

- [`bytes`](https://docs.rs/bytes/latest/bytes/)
- [`futures`](https://docs.rs/futures/latest/futures/)
- [`tokio-stream`](https://docs.rs/tokio-stream/latest/tokio_stream/)
- [`tokio-util`](https://docs.rs/tokio-util/latest/tokio_util/)
- [`tokio`](https://docs.rs/tokio/latest/tokio/)

The only external wRPC dependency not already used by Wasmtime is [`pin-project-lite`](https://docs.rs/pin-project-lite/latest/pin_project_lite/).
Other than that, wRPC depends on a few small, purpose-built crates maintained by wRPC developers and extracted from wRPC codebase itself.

## Bytecode Alliance hosted project

[wRPC] is a Bytecode Alliance hosted project and so Bytecode Alliance has full control over the project's development.

For example, if the `pin-project-lite` dependency mentioned above is an issue - that can be promptly changed.

## WIT as lingua franca

[wRPC] is built on top of WIT, which is already utilizied throughout Wasmtime codebase and, in fact, is embedded in each and every component, therefore making each component compatible with [wRPC] out of the box.
There is no need for a custom mapping scheme and/or naming convention as one would need with a different RPC protocol (e.g. gRPC).

## WebAssembly as the wire format

[wRPC] relies on [component model value definition encoding], which in turn is based on Core Wasm spec and Canonical ABI.
In fact, it can be thought of as a "packed" Canonical ABI, optimized for data size.

Usage of a familiar encoding here provides many optimization opportunities on the runtime level providing for efficient serialization and deserialization steps.

## Familiar user experience

[wRPC] provides `wit-bindgen-wrpc` binary and Rust crate, which are built on top of `wit-bindgen`, which provides a familiar user experience.
For example, `wit_bindgen_wrpc::generate!` macro supports most of the options supported by `wit_bindgen::generate!`.

Plugin developers familiar with component model would not need to learn a new technology to develop a Wasmtime plugin, rather use largely the same development flow that is already used for developing components.

## Multi-language support

Other than Rust, [wRPC] currently supports Go, but support for other languages is planned.
Language binding generators benefit greatly from prior art in `wit-bindgen`, since a lot of that logic is reused.

## Transport-agnostic

[wRPC] is transport-agnostic, with 4 supported transports at the time of writing:
- TCP
- Unix domain sockets
- QUIC
- [NATS.io](https://nats.io/)

Only TCP and Unix domain socket transports are suggested to use for Wasmtime CLI in this RFC to avoid pulling in extra dependencies.

# Open questions
[open-questions]: #open-questions

- How do plugin imports work together with host builtins? I think the right solution here is to throw an error if e.g. both built-in `wasi:keyvalue` *and* `wasi:keyvalue` plugin import are specified.
- In this RFC, there's a singleton entity providing *all* interfaces and there is no way to compose multiple plugins on CLI level for the sake of simplicity. I feel like such plugin "multiplexing" is best handled either in a "proxy" plugin or in custom Wasmtime embeddings.
- It is not clear whether machinery allowing plugins to interact with host resources is required or even a desired feature at all. For example, could a plugin export a function, which would take `wasi:http/types.fields` resource as a parameter, where `wasi:http/types.fields` is exported by the host? If so, the host would need to export `wasi:http/types.fields` methods via wRPC to allow the plugin to interact with it. I think that perhaps it's simplest to say that such use cases are not currently supported and we may want to revisit this in the future.

[wRPC]: https://github.com/bytecodealliance/wrpc
[component model value definition encoding]: https://github.com/WebAssembly/component-model/blob/main/design/mvp/Binary.md#-value-definitions
