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

* **Exception handling should be a zero-cost feature**, i.e. the
  implementation must not affect the run-time performance (execution
  speed, memory usage, etc) of programs that do not use the feature.

* **No dependence on `libunwind`.** This library provides a C API for
  determining and unwinding call chains in ELF programs. Nonetheless,
  we do not want to bring it into our trusted computing base (TCB),
  because we have found its implementations to be buggy in the past,
  and such bugs can easily compromise the integrity of Wasmtime (some
  past `libunwind` bugs:
  [#2808](https://github.com/bytecodealliance/wasmtime/issues/2808),
  [#3256](https://github.com/bytecodealliance/wasmtime/issues/3256),
  [#7997](https://github.com/bytecodealliance/wasmtime/issues/7997)).

* **Fast unwinding strategy.** Beyond the general desire for a fast
  unwinding implementation to make raising exceptions as fast as
  possible, we additionally require that the stack-walking half of
  unwinding to be fast enough to support [Wasmtime's guest
  profiler](https://docs.wasmtime.dev/examples-profiling-guest.html). Thus,
  it is important that the chosen implementation strategy does not box
  ourselves into a corner. That said, we consider it acceptable if the
  minimal viable product (MVP) of the implementation does not achieve
  peak performance as long as we retain a viable path to getting
  there.

* **Compatibility with standard formats.** Regardless of the exact
  unwinding strategy it uses internally, Wasmtime must continue to
  work with external system profilers which leverage DWARF or frame
  pointers, e.g. `perf`. Furthermore, while Cranelift may not emit
  `.eh_frame` sections directly, it should be possible that `cg_clif`
  be extended to translate the unwind data structures that Cranelift
  does generate into `.eh_frame` so that it can integrate with the
  system unwinder when implementing Rust's panics.

# Non-requirements
[non-requirements]: #non-requirements

* **No support for the legacy exception handling revision.** The
  legacy revision is being phased out.

* **No support for unwinding across host frames.** Unwinding across
  host frames would require implementing a full DWARF unwinder or
  equivalent. It also seems like it would be exceedingly likely to
  introduce complicated bugs and likely trigger undefined behavior in
  our Rust code as well. Additionally, our stack might be interleaved
  with frames from other JIT runtimes, say .NET, that might not use
  DWARF and which we will never be able to safely unwind past. Or C
  code that is compiled without unwind info and without frame
  pointers, etc...

  Note: this does not preclude throwing and catching exceptions when
  there is a host frame present in the call stack, as we can catch and
  rethrow exceptions in trampolines on the boundary.

# Proposal
[proposal]: #proposal

This section outlines the technical details of the proposal for
implementing support for exception handling in Wasmtime and Cranelift.

## CLIF Extensions
[clif-extensions]: #clif-extensions

### `ir::ExceptionTag` and `ir::ExceptionTable`

An `ir::ExceptionTag` is a newtype of an `i32`. These exception tags
are embedder-specific; Cranelift will do nothing other than forward
them into the unwinding metadata tables it returns from
compilation. It is up to the CLIF producer to coordinate with its
runtime and define the meaning of any particular exception tag.

We propose a new entity type and table of values stored in Cranelift
functions (or rather `ir::DataFlowGraph`) similar to signatures, stack
slots, etc...: the `ir::ExceptionTable` id type and its associated
`ir::ExceptionTableData` that it references. This is similar in
structure to `ir::JumpTable`, but with the following differences:

* Each entry has not only an `ir::BlockCall`, but also an associated
  `ir::ExceptionTag`.

* The first entry is not a special, default entry.

It will have Rust constructors similar to the following:

```rust
impl ir::ExceptionTableData {
    pub fn new(entries: &[(ir::ExceptionTag, ir::BlockCall)]) -> Self {
        // ...
    }
}

impl cranelift_frontend::FunctionBuilder {
    pub fn create_exception_table(&mut self, table: ir::ExceptionTableData) -> ir::ExceptionTable {
        // ...
    }
}
```

In CLIF's text format, we will show exception tables inline in the
instructions that reference them, the same way we do for jump tables,
with each entry in the form `<tag>: <block_call>`:

```
[ tag42: block3(v123), tag1312: block94, tag1337: block456(v324, v451) ]
```

### `try_call` and `try_call_indirect`

Second, we propose two new CLIF instructions:

1. A new variant of a call instruction reminiscent of LLVM's `invoke`:

   ```
   try_call func(args...), ok_label(ok_args...), [ tag: exn_label(exn_args...), ... ]
   ```

   This new instruction has the following validity requirements:

   * `func` is an `ir::FuncRef` naming a static callee.
   * `args...` match `func`'s parameter types.
   * `ok_label` and every `exn_label` must name blocks in this
     function.
   * The block parameters of `ok_label`'s block must match
     `ok_args...`.
   * The block parameters of every `exn_label` must match its associated
     `exn_args...`.

  The semantics are as follows:

  * The `func` function is invoked, passing `args...` as
    arguments.
  * When the invocation returns normally, control is transferred to the block
    named by `ok_label`, passing the `ok_args...` as block arguments.
  * When the invocation is unwound due to an exception, there are two
    sub-cases to consider:
    1. If the runtime determines that the exception matches one of the tags for
       an entry in the `ir::ExceptionTable`, then control is transferred to the
       entry's associated `exn_label` block, passing the associated
       `exn_args...` as block arguments.
    2. Otherwise, unwinding continues past this function's frame and
       control is not transferred to either the `ok_label` or the
       `exn_label`.

2. A corresponding indirect-call variant:

   ```
   try_call_indirect sig, v123(args...), ok_label(ok_args...), [ tag => exn_label(exn_args...), ... ]
   ```

   This variant has the same validity requirements and semantics as
   `try_call` but instead of taking a static `ir::FuncRef`, it takes a
   function signature immediate and dynamic function address operand
   value.

Additionally, the argument to a block call is extended from being just an
`ir::Value` to being an `enum` of either

* an `ir::Value`,
* the `i`th return of the block call's associated `try_call[_indirect]`
instruction (for normal return paths),
* or the `i`th exception payload value of the block call's associated
  `try_call[_indirect]` (for exceptional return paths).

This allows accessing a `try_call[_indirect]`'s normal and exceptional values
*only* from within the control-flow path where those values are defined. It
prevents an exception's landing pad from attempting to use the function's normal
return values, for example. The normal return values and exception payload
values are logically defined *on the control-flow edge*, so making them
accessible only within the `try_call[_indirect]` instruction's edges to other
blocks matches those semantics. The `try_call[_indirect]` instruction cannot
define these values itself, for example, because then the defs for both normal
return values and exceptional payload values would dominate -- and therefore be
"usable" from -- both the normal and exceptional successor blocks.

These new instructions have many immediates and operands. Likely more
than will naively fit in an `ir::InstructionData`. We leave the exact
details of bitpacking and encoding these instructions to the
implementation.

Note that we do not define a CLIF instruction for throwing an
exception. Instead, exception throwing must be done indirectly via an
imported function (e.g. a Wasmtime builtin libcall implemented in the
host/engine).

Furthermore, we do not define `try_return_call[_indirect]`
instructions. When you tail call out of a function, that function is
no longer on the stack to catch any exceptions unwinding the stack.

Signatures do not need to be annotated with whether they can raise
exceptions and, if so, which exceptions they raise. A regular
`call[_indirect]` that unwinds due to an exception will simply
continue unwinding over its frame. One only needs to use `try_call`
(and pay for the additional register spills) when one is catching an
exception, not for every call that might raise an exception.

We also do not define a `throw` or `raise` instruction. It is assumed
that all exceptions will be thrown by calling out to runtime routines
that invoke the unwinder directly. Thus, all CLIF instructions where
an exception could be propagated are variants of `call`
instructions. That, in turn, means that our `try_call[_indirect]`
instructions are sufficient to enumerate all exceptional control
edges.

We do not hardcode in support for two-phase exceptions (as [used by
C++](https://nicolasbrailo.github.io/blog/2013/0326_Cexceptionsunderthehood8twophasehandling.html),
for example). It should, however, be possible to build on top of this
system, so long as the first phase (the search phase) happens inside
the unwinder runtime, and Cranelift's emitted landing pads are only
ever executed once.

## Potential Implementation Strategies

There are at least two feasible implementation strategies for stack
unwinding.

1. **Side table**: we can store information about which parts of the code
   can raise exceptions and where their respective handlers are
   located.

   a. This side table can be DWARF, and Wasmtime can implement a
      bespoke unwinder for just the subset of DWARF that Cranelift
      emits, or

   b. the side table can be a completely bespoke format.

   This approach is zero-cost for the non-exceptional path.

   This approach has some complexity-at-a-distance, as the compiler
   needs to correctly emit unwind tables and the runtime needs to
   correctly interpret the tables during unwinding.

2. **Calling convention**: we can extend the calling convention to
   propagate exception information between function calls, for example
   in a dedicated register (riscv64) or a particular flags bit
   (aarch64, x86\_64, and I think s390x). If that register is non-zero
   or the bit is set, then this is an exceptional return, rather than
   a normal return, and we would branch directly to the `exn_label`
   landing pad.

   This approach imposes overhead on the non-exceptional path, even
   when the program never throws an exception.

   Correctness does not involve any coordination between distant
   components (such as the runtime's interpretation of unwind tables
   and the compiler) since the compiler can emit everything necessary
   for correctness inline.

Because the calling-convention approach imposes overhead on the
non-exceptional path, we could never turn Wasm exceptions on by
default in Wasmtime using it. That means that we would need to support
two different calling conventions (with- and without exceptions) for
every target indefinitely. And these calling conventions wouldn't just
have a different set of caller- vs. callee-saved registers or argument
and return value passing locations; they would have all this extra
check-for-and-branch-on-exception code to maintain. We consider this
scenario a non-starter. Furthermore, it doesn't make sense to use the
calling-convention approach as an incremental milestone on the way to
a side-table implementation, since the side-table implementation would
not build on top of the calling-convention implementation and would
only throw away work we previously did. We could have, instead,
dedicated that effort towards implementing the final side-table
approach in the first place. Therefore, **we will implement the
side-table approach** and we will do so right from the beginning.

We are now left with the question of whether to use side table option
(a) a DWARF subset or (b) a bespoke format. Let's start with some
observations:

* For the foreseeable moment, we do not plan to remove or replace
  Wasmtime's frame-pointer-based stack walking as part of implementing
  the Wasm exceptions proposal; unwinding exceptions will use frame
  pointers to walk the stack, the same way that existing stack walking
  happens today.[^fp-unwind-future]

* Again, at least for the foreseeable moment, we can make
  `try_call[_indirect]` instructions use a variant of the target's
  Wasm calling convention where all registers are caller-saved.

When taken together, these observations mean that our exception side
tables *only* need to encode a map from a `try_call`'s return address
PC to the set of exception tags that the `try_call` is catching and
the PC of the associated landing pad. The side tables do *not* need to
encode information about how to find the previous frame from the
current one (what DWARF calls the "canonical frame address") because
we will just use frame pointers for that. And the side tables do *not*
need to provide a mechanism for restoring callee-save registers before
control is transferred to the landing pad because there are no
callee-save registers.

Using a DWARF subset's advantages are that Wasmtime developers would
be able to reuse tools like `dwarfdump` for inspecting the side tables
in `.cwasm` files. Its downsides are that it just has way more stuff
than we need, not all of which we can just ignore. To encode the
which-exceptions-this-location-catches information, we would have to
emit Frame Description Entries (FDEs) with our custom personality
routines. But FDEs have non-optional fields, such as a list of CFI
instructions, that are used for recovering the previous frame and its
registers, which we don't need because we are always doing
frame-pointer walking. Beyond impedance mismatch cognitive overhead,
this leads to larger side tables and bloated `.cwasm` binaries.

**Therefore, in the interests of simplicity, faster lookups, and slim
`.cwasm` binaries, we will design our own bespoke format for the side
tables.** It is worth noting that we essentially do this already for
our existing traps metadata tables. We can ensure that the format is
suitable for usage straight from `mmap`, without any need to translate
the on-disk encoding into some in-memory data structure, which would
slow down `Module` creation. We can make it contain only and exactly
the information we actually need. We can use a format that is fast and
easy to decode. All this results in smaller tables, faster lookups,
and slimmer `.cwasm` binaries. While we won't be able to piggyback on
tools like `dwarfdump` &mdash; given how little information we need to
store in the side tables and the relative simplicity that brings, and
the fact that we can always use `std::fmt::Debug` in tactically-placed
logs &mdash; the lack of `dwarfdump` isn't the end of the
world.[^dwarf-future]

[^fp-unwind-future]: Whether we stop using frame pointers and start
    recovering the previous frame via a side table is an orthogonal
    question and something we can change later. On the one hand, it
    would be nice to recover the FP and get another register for
    x64. On the other hand, frame pointers make the unwinder a lot
    simpler, which is generally pretty tricky code already, and is
    also critical for correctness in the face of GC references. Frame
    pointers also make integration with system tooling (profilers,
    debuggers, etc...) dead simple and generally Just Work. But either
    way, we don't need or plan on changing anything about this at the
    moment, and can revisit this decision at some time in the future;
    don't have to worry about it now.

[^dwarf-future]: All that said, if we ever decide to switch from
    frame-pointer stack walking to side-table-based stack walking, it
    may make sense to switch to a DWARF subset at that point in time,
    since we would start using all that extra information that FDEs
    can and must encode, unlike today. Doing that switch at that
    hypothetical point in the future is fine because everything here
    is an internal implementation detail and definitely not something
    that is guaranteed across Wasmtime versions.

### Incremental Milestone: Exceptions as Uncatchable Traps

We are interested in landing this feature in a reasonably timely
manner to enable toolchain producers to use exception handling and
allow dependent proposals (such as stack switching) to run early
experiments on top of Wasmtime.

Therefore we propose to start by implementing exceptions as
uncatchable traps. This will force implementation of the initial
`wasmtime::Config` knobs, CLI flags, some Wasm-to-CLIF translation,
etc... It allows toolchains, like Kotlin, to start running subsets of
their test suite on Wasmtime, even if they use exceptions-related
instructions, as long as the dynamic execution doesn't actually throw
any exceptions.

This is also a mode that we may want to support indefinitely anyways:
some programs "must" be compiled with exceptions due to accidental
complexity with (say) their build system, but won't ever actually
raise any exceptions due to the configuration of various knobs. With
this mode, Wasmtime can run these programs and do so without the
`.cwasm` binary bloat associated with the exceptions' side tables or
the save-all-registers execution overhead of `try_call`.

Finally, it is worth noting that this mode is not visible to Wasm
guests in any way, and is therefore fully spec compliant (the same
way, e.g., Wasmtime's fuel feature is).

## Unwinding Across Instances
[unwinding-instances]: #unwinding-across-instances

It is worth noting that we must support unwinding across frames from
different modules and instances. For example, if one instance imports
and calls an exported function from a different instance, and then
that function raises an exception:

```
instance_A -> instance_B -> raise exception E
```

Note that the original caller, `instance_A`, can install an exception
handler for `E` and catch the exception that `instance_B` raises.

Our outlined implementation approach supports this scenario out of the
box. We mention it here only for completeness.

## Host APIs
[host-apis]: #host-apis

As stated in the [non-requirements] section, we do not plan to support
unwinding across host frames. However, we must allow and provide APIs
for host functions to

* inspect, catch, and propagate exceptions raised by now-unwound Wasm
  callees, and
* raise new exceptions to pass up the stack to their Wasm callers.

We propose to have our host-to-wasm trampolines catch all exceptions
raised by Wasm and return them to the host via a three-way result type
that is logically an `enum` of either a normal successful return
value, an exception, or a trap.

Similarly, we will do not plan to supporting raising Wasm exceptions
and beginning unwinding directly in the middle of arbitrary host
code. Instead, we envisage a host function will return the exception
variant of this three-way result type, potentially early-exiting via
`?`-propagation. When returning to Wasm from the host, we will unpack
that three-way result, check for the exception variant and, if found,
enter the unwinder instead of continuing with either normal
control-flow returns or the jump-directly-to-the-trap-handler
`longjmp` path.[^setjmp-longjmp]

[^setjmp-longjmp]: We do not plan on changing anything about
    Wasmtime's approach to handling traps, and its `setjmp` and
    `longjmp` usage, at this time. It is tempting to try and use the
    same mechanisms for both exceptions and traps, but it doesn't
    quite align. We don't want to use `setjmp`/`longjmp` to implement
    exceptions because that would be non-zero-cost for the
    non-exceptional path. And we don't want to reuse the outlined
    exceptions mechanism, at least for the time being, for traps
    because we want to preserve traps' *O(1)* unwinding (WASI p1
    programs, for example, implement `exit` via trap) and we need to
    jump out of signal handlers on traps, which is quite subtle and
    tricky. That said, we could revisit this decision and investigate
    sharing these mechanisms at some point in the future. It isn't
    something we must grapple with now, so for simplicity and
    expediency, we won't.

For the concrete design of the three-way result type, there are
multiple options to consider:

1. A genuine three-parameter result type capturing the three ways in
   which a Wasm program can terminate, e.g.

   ```rust
   pub enum ThreeWayResult<T> {
       Ok(T),
       Exception(wasmtime::Exception),
       Trap(wasmtime::Trap),
   }
   ```

   We would not be able to use `?` (because Rust's `Try` trait is
   currently an unstable feature) and would need to define our own
   `try!`-style macros. The resulting ergonomics for regular code that
   just propagates errors and exceptions would be poor.

2. A nested result type, e.g. `Result<Result<Success, Exception>,
   Trap>` to remain compatible with Rust's `?` operator. The first
   usage of `?` would propagate traps and, in theory, a second `?`
   would propagate exceptions. In practice, our ability to actually
   use a second `?` would be limited due to the combination of
   mismatched levels of `Result` nesting and, again, Rust's `Try`
   trait being an unstable feature. While this could sometimes provide
   better ergonomics than option (1), it wouldn't be a large
   improvement.

3. We could define an error `enum` that is either a trap or an
   exception:

   ```rust
   pub enum TrapOrException {
       Trap(wasmtime::Trap),
       Exception(wasmtime::Exception),
   }
   ```

   We would then use `Result<Success, TrapOrException>` as the result
   of running a Wasm program or calling into Wasm.

   This would allow for good `?`-propagation ergonomics. It also
   forces callers to differentiate between exceptions and traps, which
   could be either a good or bad thing depending on context and how
   often that is something callers need to care about. I think we
   could largely alleviate this with a `From<TrapOrException>`
   conversion implementation for `anyhow::Error`, or by implementing
   `std::error::Error` for `TrapOrException` and piggybacking on
   `anyhow`'s generic error-to-`anyhow::Error` conversion
   implementation.

4. The final option is to continue using `Result<Success,
   anyhow::Error>` and allow callers to try to downcast the
   `anyhow::Error` to either a `wasmtime::Trap` or a
   `wasmtime::Exception` as needed.

   This also allows for good `?`-propagation ergonomics. The primary
   difference between this and option (3) is that it is a lot easier
   to forget to check for exception versus trap, if that is something
   a caller needs to do. If it isn't something the caller needs to do,
   then it erases what would otherwise have been unnecessary cognitive
   overhead.

We propose either (3) or (4) in this RFC. The exact choice is still
somewhat of an open question, and depends on how often we think
callers will want to check for and differentiate between exceptions
and traps. The answer to that question would make the choice between
(3) and (4) clear.

# Open questions
[open-questions]: #open-questions

* We could move `try_call[_indirect]`'s `ok_label(ok_args...)` into
  `ir::ExceptionTable` if we wanted, although it is slightly funky
  since the success path is not associated with an `ir::ExceptionTag`
  the way the other entries are. But it would be a method we could
  move some data out of `ir::InstructionData` if need be. I figure we
  can finalize this kind of thing during implementation, unless others
  have strong opinions.

* How should we represent three-way results in the Wasmtime public
  API? Option (3) or (4)?

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
