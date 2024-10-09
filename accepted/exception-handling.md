# Summary

Implement support for the WebAssembly [exception handling
proposal](https://github.com/WebAssembly/exception-handling).

# Motivation
[motivation]: #motivation

The WebAssembly exception handling proposal introduces mechanisms for
raising, catching, and propagating exceptions. The proposal has been
subject to some controversy as the proposal has undergone several
significant revisions, with an early revision being deployed in
production already. As a result there effectively exists two revisions
of exception handling in the wild at the moment:

* The official W3C WebAssembly exception handling revised proposal;
* and, the legacy exception handling revision.

The revised proposal was advanced to phase 4 during [a community group
meeting in
July](https://github.com/WebAssembly/meetings/blob/main/main/2024/CG-07-16.md). Support
for exception handling is (at least) of interest to C++, Kotlin, and
OCaml toolchains. The proposal is also a prerequisite for the [stack
switching
proposal](https://github.com/WebAssembly/stack-switching/blob/main/proposals/stack-switching/Explainer.md)
which plans to use exceptions to finalise stacks. Nonetheless, to
remain standard compliant, we will eventually have to implement the
exception handling proposal in Wasmtime.

# Requirements
[requirements]: #requirements

* In its end state exception handling should be a zero-cost feature,
  i.e. the implementation must not affect the run-time performance
  (execution speed, memory usage, etc) of programs that do not use the
  feature. However, in the interim we are willing to relax this
  requirement to a "near zero-cost" such that we can use a simple
  implementation to quickly enable users to use this feature.

* No dependence on `libunwind`. This library provides a C API for
  determining and unwinding call chains in ELF programs. Nonetheless,
  we do not want to bring it into our trusted computing base (TCB),
  because we have found its implementations to be buggy in the past,
  and such bugs can easily compromise the integrity of Wasmtime (some
  past `libunwind` bugs:
  [#2808](https://github.com/bytecodealliance/wasmtime/issues/2808),
  [#3256](https://github.com/bytecodealliance/wasmtime/issues/3256),
  [#7997](https://github.com/bytecodealliance/wasmtime/issues/7997)).

* Fast unwinding strategy. We require unwinding to be fast enough to
  support [the guest
  profiler](https://docs.wasmtime.dev/examples-profiling-guest.html). Thus,
  it is important that the chosen implementation strategy does not box
  ourselves into a corner. Though, we consider it acceptable if the
  minimal viable product (MVP) of the implementation does not achieve
  peak performance for unwinding as long as there is a viable path to
  improving the performance.

* Compatibility with standard formats. Regardless of unwinding
  strategy, Wasmtime must continue to work with system profilers which
  suport DWARF or frame pointers, e.g. `perf`. Furthermore, in
  Cranelift `cg_clif` should be extended with capabilities to leverage
  the unwind info support to emit appropriate DWARF, Windows
  Structured Exception Handling, and so forth. Though, we believe a
  MVP capable of recovering the `vmctx` register should suffice for
  critical use-cases.

# Non-requirements
[non-requirements]: #non-requirements

* No support for the legacy exception handling
  revision. Justification: the legacy revision is being phased out.

* No support for unwinding across host frames. Justification:
  unwinding across host frames would require implementing a full DWARF
  unwinder or equivalent. Note: this just not preclude throwing and
  catching exceptions when there is a host frame present in the call
  stack, as we can catch and rethrow exceptions in trampolines on the
  boundary.

# Proposal sketch
[proposal]: #proposal

This section outlines the technical details of the proposal for
implementing support for exception handling in Wasmtime and Cranelift.

## CLIF extensions
[clif-semantics]: #clif-semantics

We propose two extensions to CLIF.

* A new block type `catch` which is a landing pad for exceptions. This
  type of block will have exactly one parameter which will be a
  pointer-sized integer (morally the exception value), e.g. in CLIF
  syntax
```clif
catch block123(v456: i64):
  ...
```
  We disallow regular control flow edges to `catch` blocks. Meaning
  that neither `jump`, `brif`, or `br_table` are allowed to branch
  into a `catch` block. Instead, the control flow edges to `catch`
  must come via a `try_call` instruction.

* A new call instruction `try_call <ok_label>, <exception_label>`,
  reminiscent of LLVM's `invoke`, where `<ok_label>` must be a regular
  block and `<exception_label>` must be `catch` block. The block
  parameters of `<ok_label>` must match the return types of the called
  function. The semantics is as follows: when `try_call` returns
  normally, control tranferred to the block named by `<ok_label>`;
  when `try_call` is unwound, control is tranferred to the block named
  by `<exception_label>` (note this may happen multiple times in a
  two-phase exceptions scenario).

* Potentially we may also want a `try_call_indirect <ok_label>,
  <exception_label>` for indirect calls.

We do not define a CLIF instruction for throwing an
exception. Instead, exception throwing must be done indirectly via an
imported function (e.g. a Wasmtime builtin libcall implemented in the
host/engine).

We do not hard-wire in support for two-phase exceptions. Though, it
should be possible to encode two-phase exceptions on top of the
proposed constructs. For example, the producer can emit some prelude
code that runs in the beginning of each `catch` to determine whether
the runtime is in the search phase or unwind phase.

## Implementation strategies

There are at least three feasible implementation strategies for stack
unwinding.

1. Side table: we can store information about which parts of the code
can raise exceptions and where their respective handlers are located
in a bespoke runtime-managed data structure.

2. Calling convention: we can extend the calling convention to
propagate exception information between function calls.

3. DWARF unwinder: we can implement a bespoke unwinder for a subset of
DWARF. This subset should cover the set of special-purpose registers
used by Wasmtime.

### Incremental strategy for zero-cost exceptions

We are interested in landing this feature in a reasonably timely
manner to enable producers to use exception handling and allow
dependent proposal to run experiments on top of Wasmtime. Therefore we
propose to start by implementing exceptions using calling convention
based strategy. The strategy we have in mind is not zero-cost, but
possibly cheap enough that is acceptable in the interim.

Concretely, we propose to implement non-zero cost exceptions by using
flags (e.g. the overflow flag) to determine whether a call returned
through a throw or an ordinary return instruction. Put into code:

```
call 0x12345678 # exceptional return sets the flag
jo $exception_branch
```

Before returning normally any function must clear the flag, e.g.

```
test al, al
ret
```

We reckon this approach is relatively low overhead, and it something
we can confidently implement correctly more quickly than the side
table or DWARF unwinder strategies. Adopting this strategy would allow
us to focus on the challenges of retrofitting exception support onto
CLIF and extending the Wasmtime public API first. After the fact, we
can focus the effort on getting all the nitty-gritty runtime bits for
interpreting unwind info correct.

## Unwinding across instances
[unwinding-instances]: #unwinding-across-instances

The calling convention strategy lets us aptly support unwinding across
instances, i.e. scenarios such as

```
instance_A -> instance_B -> raise exception E
```

where instance `A` has installed an exception handler for `E` and
calls into instance `B` which raises the exception `E`.

To support unwinding across instances we plan to use a callee-saved
register to hold (a pointer to) the exception `E`. This way instance
`A` can readily get hold of the exception data.

## Unwinding across host frames
[unwinding-hosts]: #unwinding-across-host-frames

As stated in the [non-requirements](:non-requirements) section we do
not plan to support unwinding across host frames. However, to
gracefully handle exceptions that attempt to cross wasm-host boundary
we propose to have our host-to-wasm trampoline catch any Wasm-thrown
exceptions and return them via a three-way result type.

Similarly, we will do not plan to supporting raising Wasm exceptions
directly in host code. Instead, we envisage a host function supply an
element of the exception type as an argument, which the wasm-to-host
trampolines will reify as a genuine exception and raise inside the
Wasm code.

For the design of the three-way result type there are multiple options
to consider:

* A genuine three-parameter result type capturing the three ways in
  which a Wasm program can terminate, e.g. `Result<Success, Exception,
  Trap>`.
* A nested result type, e.g. `Result<Result<Success, Exception>,
  Trap>` to remain compatible with Rust's `?` operator.
* Or continue to use `Result<Success, anyhow::Error>` and allow
  callers to try to downcast the `anyhow::Error` to either a trap or
  an exception as needed.

# Open questions
[open-questions]: #open-questions

* We need to decide whether `catch` blocks in CLIF are allowed to use
  values from dominating blocks.
* How should we represent three-way results in the Wasmtime public
  API?

# References
[references]: #references

* [Exception handling in LLVM](https://llvm.org/docs/ExceptionHandling.html)
* [Itanium C++ exception handling ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)
* [Framehop stack unwinder](https://github.com/mstange/framehop)
* [Reliable and fast DWARF-based stack uwinding](https://inria.hal.science/hal-02297690/document)
* [How fast can CFI/EXIDX-based stack unwinding be?](https://blog.mozilla.org/jseward/2013/08/29/how-fast-can-cfiexidx-based-stack-unwinding-be/)
* [Unwinding stack by hand with frame pointers and ORC](https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc)
* [The return of frame pointers](https://www.brendangregg.com/blog/2024-03-17/the-return-of-the-frame-pointers.html)
* [The SFrame format](https://sourceware.org/binutils/docs/sframe-spec.html)