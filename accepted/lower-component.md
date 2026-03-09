# Summary
[summary]: #summary

This document proposes to create a pair of tools to support running components
using any compatible WebAssembly.  These tools could be used either as an
alternative to providing native support for components in a given runtime or as
a temporary polyfill while native support is being built for that runtime.

# Motivation
[motivation]: #motivation

Implementing the complete Component Model specification is a non-trivial task
which entails significant up-front effort as well as ongoing maintenance.  [Jco]
has proven useful as a polyfill for JS-embedded Wasm runtimes which don't yet
have native component support, but it entails a performance penalty and doesn't
support stand-alone Wasm runtimes.

On the other hand, though Wasmtime has performant, native support for
components, that code cannot easily be reused for other runtimes.  In addition,
that implementation significantly expands the trusted compute base which must be
audited for correct and secure behavior beyond what is needed for core Wasm.  It
is closely integrated with the unsafe internals of the runtime, requiring
specialized knowledge and care to maintain and modify.

Ideally, a component implementation would provide the best qualities of both of
those implementations, while addressing or side-stepping their weaknesses:

- Portable to arbitrary runtimes (JS-embedded or standalone)
- Performant
- Secure, e.g. doing as much work (and allocation) as practical in sandboxed
  guest code, minimizing the TCB
- Maintainable without specialized knowledge of the internals of a particular
  runtime
- Compatible with embedded and/or memory-limited scenarios

[Jco]: https://github.com/bytecodealliance/jco

# Proposal
[proposal]: #proposal

This proposal includes four things:

- A `lower-component` tool which takes a component as input and "lowers" it into
  a core module
- A C API representing the intrinsics which a host runtime must provide in order
  to run a module produced by `lower-component`
- A `host-wit-bindgen` tool which takes a WIT world and produces code for a
  chosen target language to instantiate a `lower-component`-generated module and
  invoke its exports
- A C API representing the operations which a host runtime must provide to
  enable instantiation, invocation, access to memories and globals, etc. for
  `host-wit-bindgen`-generated code to make use of

## `lower-component`

The job of this tool is to take an arbitrary component as input and "lower" it
into a core module which may be run using an arbitrary Wasm runtime, assuming
the host provides a small set of intrinsics (covered later) which the module
will call as import functions.  This tool could either be used ahead-of-time or
just prior to instantiation.

In general, a component may include a composition of more than one subcomponent,
each instantiation of which may require its own memory and table.  In that case,
the output module will use multiple memories and tables and include generated
adapter code to "fuse" the imports of one component to the exports of another,
handling validation, cross-memory copies, etc., just as Wasmtime's FACT does
today.

In addition to the generated "fused adapter" code, the output module will
include component model runtime code, separately compiled from Rust source,
which handles, among other things:

- table management for resource and waitable values
- guest-to-guest stream and future I/O
- task and thread bookkeeping

That runtime code will itself make use of intrinsic functions imported from the
host in order to do things only the host can do, e.g. create, suspend, and
resume fibers and collect error backtraces.  See the next section for details.

In the case of component-level exports which involve stream and/or future types,
the generated module would include function exports which the host may call to
create new values of those types.  This is necessary because the tables for such
values are managed internally by the guest, not by the host.

### Multiply-instantiated Modules

One challenge with lowering arbitrary components is that a component may
instantiate the same module more than once.  In that case, we have three options:

- Reject the component
- Generate an output module with duplicate copies of each function in a
  multiply-instantiated module, one per instance.
    - Note that leaf functions which do not use memory or globals can be reused
      without duplication.
    - This could lead to significant bloat for "batteries-included" guest
      languages like Python which do not have dead-code-elimination.
- Generate multiple output modules, plus metadata indicating how to instantiate
  and link them together.
    - This would require specifying how that metadata is represented and how the
      whole combination of modules+metadata should be packaged.
  
## Host C API for Lowered Components

As mentioned above, modules produced using `lower-component` can't (yet) express
all operations in core Wasm, and therefore must use intrinsics for certain
things:

- creating, suspending, and resuming fibers
- reading and writing fiber-local state
- generating stack traces for component-level errors

Fiber management could be expressed using the Stack Switching proposal, and
indeed `lower-component` will likely have an option to use those instructions,
but that proposal is not yet widely implemented, so we use intrinsics for
maximum portability.  Hopefully all of the above features will eventually be
covered by widely-supported core Wasm instructions.

Note that these intrinsics need not be implemented in C, nor a language with
native support for the Wasm C ABI; we simply use C as a way to represent an ABI
in a familiar, human-readable format.

```
(TODO: Sketch the proposed API)
```

## `host-wit-bindgen`

This tool takes as input a WIT world and produces source code for a given target
language which may be used to define component-level host functions, instantiate
`lower-component`-produced modules, and invoke its exports, etc.

Similar to `wit-bindgen`, this could be packaged either as a single tool
supporting multiple target languages or as separate tools, one per language.  In
either case, the generated code would bottom out in calls to the runtime as
defined by the API described in the next section.

Alternatively, the functionality of `host-wit-bindgen`-generated code could be
provided by a library providing a general-purpose, dynamic API for creating
component values, defining host functions, and calling functions.  This would be
useful in scenarios where the shape of the component is not known ahead of time,
and/or the target language is already so dynamic that code generation is
redundant.

## Host C API for Embedder Bindings

In theory, `host-wit-bindgen` could support multiple front-ends (e.g. Rust,
Python, C#, Go, etc.) _and_ multiple back-ends (Wasmtime, WAMR, Wazero, JS,
etc.), but it's probably easier to define a runtime-agnostic C API which each
runtime can implement to support the low-level operations required by
`host-wit-bindgen`-generated code.  Those operations include:

- creating a "store" in which one or more modules may be instantiated
- defining host functions
- instantiating a module
- calling a module's exports
- reading from and writing to a module's memories and globals
- creating, suspending, and resuming fibers
- generating stack traces
- reading and writing fiber-local state

Given that a C API doesn't make sense in e.g. a web browser, this could be
mirrored as a JS API for use in JS-embedded runtimes.

```
(TODO: Sketch the proposed API)
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

See the `Motivation` section above for rationale.

At a high level, the alternative to creating a runtime-agnostic component model
implementation is to implement a native one for each runtime, possibly with some
parts factored out into reusable libraries.  This is the approach we've taken so
far with Wasmtime, although not necessarily the one we'd choose with the benefit
of hindsight.  In any case, a runtime-agnostic implementation would be useful as
a temporary polyfill for use in a given runtime until a native implementation is
complete.

## Prior art

There are already a few projects which polyfill the component model:

- [Jco](https://github.com/bytecodealliance/jco) for JS-embedded runtimes
- [Gravity](https://github.com/arcjet/gravity) for Wazero on Go
- [Meld](https://github.com/pulseengine/meld) for arbitrary runtimes

# Open questions
[open-questions]: #open-questions

- How to handle multiply-instantiated modules, and how common are the components
  which do that?
- Who should be responsible for type checking during host->guest and guest->host
  invocations?
    - If `host-wit-bindgen`, where will in the flattened module will it find the
      type metadata it needs?
    - If `lower-component`, it could be harder to optimize (e.g. doing it on
      every call, whereas `host-wit-bindgen` could lean on the target language's
      static type guarantees, as `wasmtime-wit-bindgen` does today)
- How much thread-local state management can be handled by the
  `lower-component`-generated module vs. by the host?
