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

3. **Risk of Enabling Standards Circumvention:** While none of these
   alternatives deviate from Wasm semantics, there does exist the potential risk
   of making this new functionality powerful enough that it can be used as a
   vector for circumventing standard Wasm. We do not want to entice developers
   to target their whole applications to compile-time builtins, rather than
   standard Wasm itself, so as to gain access to privileged intrinsics.

More details on each approach will follow, but for the purpose of quickly
reviewing their tradeoffs, I've summarized our three alternatives along these
additional dimensions in the following table:

| Candidate Solution | API Burden | Implementation Burden | Risk of Enabling Standards Circumvention |
|---|---|---|---|
| Expose CLIF | ⭐★★★★ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐★ |
| Mini-Language | ⭐⭐⭐⭐⭐ |  ⭐⭐⭐★★ | ⭐⭐⭐⭐⭐ |
| Self-hosted Wasm | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐★★★★ through ⭐⭐⭐⭐⭐ ? |

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

The risk of enabling standards circumvention seems relatively low, since writing
whole programs with `cranelift_frontend::FunctionBuilder` would be unpleasant,
to say the least.

Finally, it does not seem possible to support Winch with this approach. Maybe
that is okay, since if you care about runtime speed, you should be using
Cranelift anyways.

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

The risk of enabling standards circumvention is very low: we define this
mini-language and we can, again, simply avoid making it powerful enough to
target for whole applications.

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

code_builder.define_compile_time_builtins(
    "foo",
    r#"
        (module
            (import "__wasmtime_intrinsics" "i32.load_native" (func (param i64) (result i32)))
            (import "__wasmtime_intrinsics" "i32.store_native" (func (param i64) (result i32)))

            ;; ...

            (func (export "bar") (param i32 i32) (result i32)
                (i32.add (local.get 0) (local.get 1))
            )
        )
    "#,
)?;
```

The API burden is minimal: WebAssembly is already precisely defined and we can
take Wasm binaries as input rather than provide a builder with a bunch of
methods.

The maintenance burden is also minimal: we must already parse and generate code
for Wasm.

Unfortunately, the risk of enabling standards circumvention is quite high if we
are not careful. If we implemented this approach naively, we could accidentally
create a "Wasm prime" target that is just Wasm but with non-standard extensions
for escaping the sandbox and ambiently accessing capabilities. That said, if we
put some restrictions on the shape of the Wasm that implements a compile-time
builtin, for example what state it can define and imports it can access, then I
think we can reduce this risk (more on this later).

The final benefit that this approach brings is that it is potentially a stepping
stone towards general Wasm function inlining in Wasmtime.[^wasm-inlining]

[^wasm-inlining]: Generally, a Wasm module was produced by LLVM and potentially
    optimized by `wasm-opt`, and all of the opportunities for beneficial
    inlining have already been taken. That is no longer true in a components
    world, where we are linking multiple core Wasm modules together, each of
    which were compiled independently, and can now see which exports are wired
    up to which imports.

# Proposal

As for how to define compile-time builtins, this RFC proposes that we move
forward with the Wasm self-hosting approach.

First, we add a `wasmtime::CodeBuilder::define_compile_time_builtins` method:

```rust
impl CodeBuilder {
    pub unsafe fn define_compile_time_builtins(
        &mut self,
        namespace: &str,
        self_hosted_wasm: &[u8],
    ) -> Result<&mut Self> {
        // ...
    }
}
```

`self_hosted_wasm` is a Wasm binary (or WAT if the `"wat"` cargo feature is
enabled). We validate that it fits within the restrictions we impose upon
self-hosted Wasm.

Upon successful return, if the code that the `CodeBuilder` is building imports
from `namespace` a function with the name of one of `self_hosted_wasm`'s
exports, then we will do the following:

* Validate that the import and export type signatures match
* Erase that import from the builder's resulting `Module` or `Component`
* Inline the body of the builtin function at all call sites

Note that we still have to define a standalone function inside the compiled
artifact for each compile-time builtin that gets used just in case it is ever
`ref.func`ed or re-exported.

## Restrictions on Self-Hosted Wasm

Self-hosted Wasm that is defining compile-time builtins will not be allowed to
define any additional state:

* No memories
* No globals
* No tables

Compile-time builtins will not (at least initially, see open questions) be able
to import anything other than functions from the `__wasmtime_intrinsics`
namespace nor to make any function calls, other than to those imported
intrinsics.

`ref.func`ing or re-exporting imported intrinsics is not allowed.

These constraints should retain the flexibility needed to implement simple
getters and setters inline while also preventing this environment from being
something that general applications can target (which would risk enabling the
circumvention of standard Wasm).

We will need to implement a validation pass (or validate as we inline) that
checks these properties.

## Intrinsics

This is a list of the intrinsics that will be available for compile-time
builtins under the `__wasmtime_intrinsics` import namespace. Calls to intrinsics
do not become actual function calls, they are replaced with a handful of native
instructions.

First up are load and store operations for the native address space. The
`pointer` type is either an `i32` or `i64` depending on the target's pointer
width. The `native` mnemonic means that the memory operation is operating on the
native memory address space, not a particular Wasm memory, and uses native
endianness. The `_s` and `_u` mnemonics have the same meaning as in core Wasm:
they specify whether the value is signed- or unsigned extended. The `8` and `16`
suffixes also have the same meaning as in core Wasm: only store the bottom N
bits of the value to memory.

* `i8.native_load_s: [pointer] -> [i32]`
* `i8.native_load_u: [pointer] -> [i32]`
* `i16.native_load_s: [pointer] -> [i32]`
* `i16.native_load_u: [pointer] -> [i32]`
* `i32.native_load: [pointer] -> [i32]`
* `i64.native_load: [pointer] -> [i64]`
* `i32.native_store8: [pointer i32] -> []`
* `i32.native_store16: [pointer i32] -> []`
* `i32.native_store: [pointer i32] -> []`
* `i64.native_store: [pointer i64] -> []`

Next, we have an intrinsic for getting the address of the host data in the
resource table for a particular `i32` resource handle. It will raise a trap on
out-of-bounds resource table accesses.

* `resource.address: [i32] -> [pointer]`

Note that we do *not* define intrinsics for directly accessing or addressing the
`vmctx`, any linear memories, or any other internal state of the Wasm
instance. While Wasmtime needs those abilities to implement operations like
`global.get`, compile-time builtins do not get to see inside Wasmtime's
implementation details. They are only given helpers for accessing things that
the embedder themselves defined, such as the elements inside a resource
table.[^blah]

[^blah]: Of course, although they should only ever access embedder-defined data
    as part of their safety contract, compile-time builtins ultimately have the
    capability to access absolutely anything -- including Wasmtime internals --
    since they have native address space loads and stores. That doesn't mean we
    need to add more footguns than we fundamentally must.

While we needn't implement all of these intrinsics from the very start, we
should in the fullness of time implement all of them.

# Open questions
[open-questions]: #open-questions

* Is there some other approach to defining compile-time builtins for inlining
  that I am failing to identify and which is better than anything proposed here?

* How do we specify which resource table we want to access in the
  `resource.address` intrinsic? Do we need separate mechanisms for defining core
  and component compile-time builtins, where component builtins are self-hosted
  components and core builtins are self-hosted core modules?

* Should we even support defining compile-time builtins with Winch at all? Or
  should it always make the function calls at runtime, since if you wanted to
  prioritize runtime speed you'd be using Cranelift anyways?

* Should we allow self-hosted Wasm to import and call non-intrinsic functions?

  This would allow implementing fast paths inline, but with out-of-line
  fallbacks for slow paths. Imagine a resource that represents a host `Vec<u8>`:
  the path for `push`ing when there is capacity can all be done inline, but
  still call out to a host function when there isn't capacity and the vec must
  be resized. This raises all sorts of design questions though:

  * How do we define and validate what functions are available to a compile-time
    builtin at compile time? Do they have to be a subset of the core module's
    imports?
  * How do we define these slow-path functions in a `Linker`, or in
    `Instance::new`, such that they are only accessible to the compile-time
    builtin and not the rest of the module?
  * Or does this case not erase the import at compile time, but instead only
    actually end up calling the import if the inline code failed to "satisfy"
    the call?

  I think this is something we will want eventually, but I'd like to get
  something basic working first before tackling this more-complicated use case.
