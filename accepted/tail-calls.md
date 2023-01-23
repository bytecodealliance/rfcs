* [Summary][summary]
* [Motivation][motivation]
* [Requirements][requirements]
* [Non-Requirements][non-requirements]
* [A Review of Our Existing Trampolines, Calling Conventions, and Call Paths][review]
  * [Calling Conventions][review-cc]
  * [Trampolines and Call Paths][review-trampolines]
* [Proposal][proposal]
  * [New CLIF Instructions][proposal-instructions]
  * [New Wasm Calling Conventions in Cranelift][proposal-calling-conventions]
  * [New Trampolines and `VMCallerCheckedAnyfunc` Changes][proposal-trampolines]
  * [Incremental Implementation Plan][incremental-plan]
* [Rationale and Alternatives][rationale-and-alternatives]
* [Open Questions][open-questions]

# Summary
[summary]: #summary

Add support for the WebAssembly tail calls proposal to Wasmtime by

* adding new `return_call` and `return_call_indirect` CLIF instructions (similar
  to the Wasm instructions),
* replacing the `wasmtime_*` calling conventions in Cranelift with new `wasm`
  calling convention where callees clean up their caller's stack space
  (e.g. stack arguments),
* and redesigning Wasmtime's system of trampolines between Wasm and the host
  such that we never chain trampolines.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

In general, we want Wasmtime to support well-designed, standards-track
WebAssembly proposals like the tail calls proposal. The proposal (currently in
phase 4) is mature, stable, and implemented in V8 and JSC.

A variety of Wasm producers will and already do target Wasm tail calls. In
`clang`, C++20's coroutines lower to calls annotated with LLVM's `musttail`
attribute, which fails to compile to Wasm unless the tail calls proposal is
enabled. LLVM's optimizer will additionally turn regular calls into tail calls
when the proposal is enabled. Compiling to Wasm with a continuation passing
style where everything is a tail call is a natural fit for some domains, such as
lowering a finite state machine to Wasm. Many functional languages guarantee
proper tail calls and have to bend over backwards to deliver them without the
Wasm tail calls proposal.

So whether we want to support tail calls is pretty uncontentious. What is more
subtle and up for debate is the choice of how we implement the proposal and the
trade offs we make in the implementation's design. But before I can argue why
this RFC's trade offs represent the best point in the design space, we need to
enumerate our requirements (which are sometimes in tension with one another).

# Requirements
[requirements]: #requirements

1. Wasmtime must continue to be able to load and call into pre-compiled Wasm
   modules when built without a compiler. That is, calling into Wasm cannot
   require on-demand JIT compilation.
2. Function calls (both Wasm to Wasm and crossing the boundary between the host
   and Wasm) should be as fast as possible.
3. The Wasmtime API must continue to support wrapping host functions into Wasm
   functions.
4. The Wasmtime API must continue to support calling arbitrary public Wasm
   functions. This includes calling Wasm re-exports of imported functions that
   happen to be backed by host functions.
5. Any function can be tail called and we don't know which will or won't be tail
   called ahead of time. Even if a module doesn't use tail calls itself, another
   module can import that module's exports and tail call them.
6. Tail calls between Wasm functions (even across different Wasm modules!) must
   use constant stack space.

In general, we need at least one (and ideally at most one) trampoline to call
into and out from Wasm. This is already true today: we use trampolines to keep
track of the stack pointer, frame pointer, and program counter registers on
entry to and exit from Wasm for fast stack walking. Supporting tail calls will
require extending our trampolines' responsibilities to include converting
between our native calling conventions (that don't have general tail calls
support) to another calling convention that does support tail calls. But since
we cannot JIT trampolines on demand, we have to get them from somewhere
else. Where trampolines come from, who generates them, and when that happens are
some of the central concerns that this RFC aims to resolve while simultaneously
balancing the concerns above.

# Non-Requirements
[non-requirements]: #non-requirements

1. Tail calls to host functions do not need to guarantee constant stack space.

# A Review of Our Existing Trampolines, Calling Conventions, and Call Paths
[review]: #a-review-of-our-existing-trampolines-calling-conventions-and-call-paths

Before we propose changes to our calling conventions, trampolines, and the code
paths implementing function calls between the host and Wasm, it is important we
understand the current state of these things in Wasmtime.

## Calling Conventions
[review-cc]: #calling-conventions

The host can use two calling conventions when calling into Wasm, depending on
the Wasmtime API used to make the call:

1. **The native calling convention.** The native calling convention, e.g. the
   System V ABI's calling convention on Linux.

2. **The "array" calling convention.** This is a calling convention where
   arguments and return values are passed via a dynamic array. Every logical
   Wasm function signature in this calling convention has the same actual
   native-calling-convention signature where the value array is passed as two
   arguments: a pointer and a length.

   All Wasmtime APIs to create a function that uses this calling convention
   (i.e. `wasmtime::Func::new`) are currently feature-gated on the Wasmtime
   build having a Wasm compiler available. This RFC does not propose to alter
   this feature-gating. So, while we generally cannot rely on JIT compilation,
   whenever the callee is using the array calling convention we can.

Our Wasm uses only one calling convention[^one-wasm-calling-convention]:

1. **The `wasmtime_*` calling convention.** We took care to design our
   `wasmtime_*` calling conventions (such as `wasmtime_system_v`) in Cranelift
   to be identical to the host's native calling convention for Wasm function
   signatures. Arguments to and return values from Wasm are passed in the same
   registers or stack slots as they are for native functions. This means that
   the Wasm and native calling conventions are *currently* interoperable.

[^one-wasm-calling-convention]: Our Wasm actually uses one calling convention
    *per target*, so this is technically more than one calling convention, but
    we can treat it as a single calling convention for our current purposes.

## Trampolines and Call Paths
[review-trampolines]: #trampolines-and-call-paths

We have a variety of ways to call into Wasm from the host and call back out to
the host from Wasm but fundamentally, regardless which Wasmtime API or Wasm
`call` instruction you're using, Wasmtime has to support the cross product of
calls with each of our calling conventions for callers and callees:

```
caller: {wasm, native, array} ⨯ callee: {wasm, native, array} = {
    wasm → wasm,   wasm → native,   wasm → array,
    native → wasm, native → native, native → array,
    array → wasm,  array → native,  array → array
}
```

In general, the way our current implementation works is that all callers and
callees agree to meet in a single calling convention: our Wasm calling
convention and the subset of the native calling convention that is interoperable
with it. We have an *entry* trampoline, used when the host calls into Wasm, and
an *exit* trampoline, used when Wasm calls out to the host. These trampolines
record registers that allow us to implement fast stack walking in Wasmtime. This
means that all calls that cross the boundary between the Wasm and the host will
have at least one trampoline. Wasm to Wasm calls don't cross that boundary, and
already speak the same calling convention, so they don't need any
trampolines. If either the caller or callee doesn't actually speak that common
calling convention (e.g. uses the array calling convention) then we chain
together additional trampolines so that the function can pretend to speak the
Wasm calling convention and the other side is none the wiser. However, this can
result in situations where both sides speak the same non-Wasm calling
convention, but are pretending to speak the Wasm calling convention, and we end
up chaining multiple trampolines together that unnecessarily shuffle arguments
and returns to the Wasm calling convention's positions and then back again.

I'll now dig into the code paths, APIs, and trampolines involved for each of the
cases from our cross product. I've provided diagrams where the blocks are
callers, callees, and trampolines and the arrows are calls and returns. The
arrows are labeled with the calling convention of the associated call or return.

### `wasm → wasm`

This is Wasm calling another Wasm function. We only need trampolines when
crossing the boundary between the host and Wasm, which these calls don't do,
therefore we don't go through any trampolines or additional indirections in this
scenario.

This is a happy path!

```
+------+                +------+
|      |---wasm-call--->|      |
| Wasm |                | Wasm |
|      |<--wasm-return--|      |
+------+                +------+
```

### `wasm → native`

This is Wasm (whether via `call`, `call_indirect`, or eventually `call_ref`)
calling a host function that was defined via the `wasmtime::Func::wrap` API or
similar.

The calling conventions are (currently) interoperable, so arguments and returns
don't need any converting or shuffling.

However, because we are crossing the boundary between the host and Wasm, we need
a trampoline to record the Wasm's exit stack pointer, frame pointer, and program
counter registers so Wasmtime can do fast and accurate stack walking. This
trampoline is carefully hand-written in assembly to only use volatile,
caller-saves scratch registers and to not use any stack space since that could
perturb the expected locations of arguments passed on the stack. Because it
cannot push to the stack, it cannot use a normal call instruction to transfer
control to the actual callee, it uses a tail call. Furthermore, because this one
trampoline is reused for all callees, it does an indirect tail call to the
callee. The callee is loaded out of the `VMHostFuncContext::host_func` field.

```
+------+                                                    +------+
|      |              +-------------+                       |      |
| Wasm |--wasm-call-->| Asm. Tramp. |--indirect-tail-call-->| Host |
|      |              +-------------+                       |      |
|      |                                                    |      |
|      |<------------------native-return--------------------|      |
|      |                                                    |      |
+------+                                                    +------+
```

### `wasm → array`

This is Wasm (whether via `call`, `call_indirect`, or eventually `call_ref`)
calling a host function that was defined via the `wasmtime::Func::new` API.

The caller first calls to the same hand-written assembly trampoline as the
`wasm → native` case, but then that tail calls not to the actual callee but to
another trampoline.

The second trampoline is responsible for translating arguments from the
`wasmtime_*` calling convention to the array calling convention (that is,
allocating stack space and spilling the arguments to that stack space), calling
the actual host callee function, and then unspilling the return values out of
the on-stack array and into the locations that the `wasmtime_*` calling
convention prescribes.

Note that the `wasmtime::Func::new` API is feature-gated on the Wasmtime build
including a compiler, so in this scenario, we *can* rely on JIT compilation. The
array trampoline is JIT'd when the host function is created if we don't already
have a trampoline for its function signature.

This scenario is currently suboptimal: we have two trampolines where we should
really only have one.

```
+------+                                                    +--------------+                 +------+
|      |              +-------------+                       |              |                 |      |
| Wasm |--wasm-call-->| Asm. Tramp. |--indirect-tail-call-->| Array Tramp. |---array-call--->| Host |
|      |              +-------------+                       |              |                 |      |
|      |                                                    |              |                 |      |
|      |<-------------------wasm-return---------------------|              |<--array-return--|      |
|      |                                                    |              |                 |      |
+------+                                                    +--------------+                 +------+
```

There is also a second kind of call where we have this pair of caller and callee
calling conventions: when a Wasm component calls a lowered host function. This
scenario does not rely on, or have access to, JIT compilation. It does not use
the hand-written assembly trampoline either. When the component is compiled, a
trampoline for each function import is compiled along with it. The trampoline
for lowered function imports does similar spilling and unspilling from a values
array as the previous trampoline did, and it does some interface types-specific
stuff, but it *also* saves the Wasm's exit registers for stack walking.
Therefore, we don't need the hand-written assembly trampoline and this scenario
avoids the suboptimal double-trampoline situation.

```
+-----------+                 +------------+                     +------+
|           |                 |            |                     |      |
| Component |----wasm-call--->| Lowered    |-----native-call---->| Host |
|           |                 | Function   |                     |      |
|           |<--wasm-return---| Trampoline |<---native-return----|      |
|           |                 |            |                     |      |
+-----------+                 +------------+                     +------+
```

### `native → wasm`

We've now completed enumerating all the ways that Wasm can call out to the host,
and begin enumerating the ways that the host can call into Wasm.

In this first host-to-Wasm call scenario, the caller is using the
`wasmtime::TypedFunc::call` API and the callee is a Wasm function. Because the
calling conventions are interoperable, we don't need to shuffle or convert
arguments and return values.

Because we are crossing the boundary between the host and Wasm, we need a
trampoline. This trampoline is very similar to the one used for `wasm → native`
calls. It is also hand-written assembly that works for all function signatures
and it also does an indirect tail call to the actual callee. It loads its callee
out of the `VMContext::callee` field and Wasmtime's internal call paths need to
take care to properly initialize that field when calling into Wasm. Rather than
saving exit registers for stack walking, it saves the entry registers.

```
+------+                                                      +------+
|      |                +-------------+                       |      |
| Host |--native-call-->| Asm. Tramp. |--indirect-tail-call-->| Wasm |
|      |                +-------------+                       |      |
|      |                                                      |      |
|      |<--------------------wasm-return----------------------|      |
|      |                                                      |      |
+------+                                                      +------+
```

### `native → native`

This is a subtle scenario that can be kind of tricky. Both the caller and callee
are using the native calling convention, which means the callee is using the
`wasmtime::TypedFunc::call` API and the callee was defined with
`wasmtime::Func::wrap`. This call doesn't technically require any trampolines,
since we aren't actually crossing the boundary between the host and Wasm!
However, that is a dynamic property of the `wasmtime::TypedFunc::call`
invocation, and we don't want to have branches for this edge case in that code
since it is pretty rare and we want this code to be as easy to maintain as
possible and as tight as we can have it for the common case. So instead the
caller uses the same trampoline it would use when calling into Wasm, the callee
uses the same trampoline it would use when calling out to the host, and we end
up in another suboptimal, double-trampoline scenario.

We also end up recording an empty range of "Wasm" on the stack for our
stack-walking machinery, which doesn't affect correctness, but does minorly
affect performance (at least in theory, I suspect it is so small it would be
hard to actually measure).

```
+------+                                                                                            +------+
|      |                +-------------+                       +-------------+                       |      |
|      |                | Caller      |                       | Callee      |                       |      |
| Host |--native-call-->| Asm. Tramp. |--indirect-tail-call-->| Asm. Tramp. |--indirect-tail-call-->| Host |
|      |                +-------------+                       +-------------+                       |      |
|      |                                                                                            |      |
|      |<---------------------------------wasm-return-----------------------------------------------|      |
|      |                                                                                            |      |
+------+                                                                                            +------+
```

### `native → array`

This scenario is similar to the last: the caller is again using the
`wasmtime::TypedFunc::call` API but this time the callee is a host function that
was defined with `wasmtime::Func::new` rather than `wasmtime::Func::wrap`. The
set of trampolines starts out identical, but then we add one more to convert
arguments from the native/wasm calling convention to the array calling
convention and then back again for return values. This is the same trampoline
from the `wasm → array` scenario that is JIT compiled if necessary. We end up
with an embarrassing chain of three trampolines.

```
+------+                                                                                            +--------+                 +------+
|      |                +-------------+                       +-------------+                       |        |                 |      |
|      |                | Caller      |                       | Callee      |                       | Array  |                 |      |
| Host |--native-call-->| Asm. Tramp. |--indirect-tail-call-->| Asm. Tramp. |--indirect-tail-call-->| Tramp. |---array-call--->| Host |
|      |                +-------------+                       +-------------+                       |        |                 |      |
|      |                                                                                            |        |                 |      |
|      |<---------------------------------wasm-return-----------------------------------------------|        |<--array-return--|      |
|      |                                                                                            |        |                 |      |
+------+                                                                                            +--------+                 +------+
```

### `array → wasm`

In this scenario, the caller is using the "untyped" `wasmtime::Func::call` API
to pass in an array of `wasmtime::Val` arguments and the callee is a Wasm
function.

We use the same trampoline for recording entry registers for stack walking that
is hand-written in assembly. However, before we call the actual Wasm callee we
need to additionally spill the arguments from the array to their Wasm calling
convention locations, and after callee returns we need to do the opposite with
the return values. This is done by the `host_to_wasm_trampoline` Rust function
that is monomorphized for our various Wasm function signatures and is defined
within the `macro_rules!` macro that also defines our `IntoFunc` trait
implementations. Since this trampoline is authored in Rust, it uses the native
calling convention to call the actual Wasm callee, which works because we
currently take care to make sure that the calling conventions are interoperable.

```
+------+                                                     +--------+                +------+
|      |                                                     |        |                |      |
|      |               +-------------+                       | Array  |                |      |
| Host |--array-call-->| Asm. Tramp. |--indirect-tail-call-->| Tramp. |--native-call-->| Wasm |
|      |               +-------------+                       |        |                |      |
|      |                                                     |        |                |      |
|      |<------------------------array-return----------------|        |<--wasm-return--|      |
|      |                                                     |        |                |      |
+------+                                                     +--------+                +------+
```

### `array → native`

Here, the caller is a host function defined with the "untyped"
`wasmtime::Func::call` API and passes its arguments and returns via a slice of
`wasmtime::Val` values. The callee is also a host function, but was defined with
the `wasmtime::Func::wrap` API and uses the native calling convention. The
result ends up being identical to the `array → wasm` case right above.

```
+------+                                                      +--------+                  +------+
|      |                                                      |        |                  |      |
|      |                +-------------+                       | Array  |                  |      |
| Host |--native-call-->| Asm. Tramp. |--indirect-tail-call-->| Tramp. |---native-call--->| host |
|      |                +-------------+                       |        |                  |      |
|      |                                                      |        |                  |      |
|      |<------------------------native-return----------------|        |<--native-return--|      |
|      |                                                      |        |                  |      |
+------+                                                      +--------+                  +------+
```

### `array → array`

In this case we have a caller using the `wasmtime::Func::call` API on a callee
that was defined via `wasmtime::Func::new`.

Although the caller and callee are using the same calling convention, both are
chaining trampolines to pretend that they speak the Wasm calling convention. The
result is a tiny car that holds way too many clowns. We have a whopping four
trampolines: the hand-written assembly trampoline for entry into Wasm, the
array-to-native trampoline, the hand-written assembly trampoline for exiting
Wasm, and the native-to-array trampoline.

Luckily these calls are rare and not generally performance sensitive (otherwise
callers would be using `wasmtime::TypedFunc` and callees would define their
functions with `wasmtime::Func::wrap`).

```
+------+                                                     +--------+                                                        +--------+                 +------+
|      |               +-------------+                       |        |                  +-------------+                       |        |                 |      |
|      |               | Caller      |                       | Array  |                  | Callee      |                       | Array  |                 |      |
| Host |--array-call-->| Asm. Tramp. |--indirect-tail-call-->| Tramp. |---native-call--->| Asm. Tramp. |--indirect-tail-call-->| Tramp. |---array-call--->| Host |
|      |               +-------------+                       |        |                  +-------------+                       |        |                 |      |
|      |                                                     |        |                                                        |        |                 |      |
|      |<------------------------------array-return----------|        |<---------------native-return---------------------------|        |<--array-return--|      |
|      |                                                     |        |                                                        |        |                 |      |
+------+                                                     +--------+                                                        +--------+                 +------+
```

# Proposal
[proposal]: #proposal

<!-- The meat of the RFC. Explain the proposal in sufficient detail to support -->
<!-- building consensus around the primary design questions and how they affect -->
<!-- stakeholders. The fine details of a design will be finalized during -->
<!-- implementation review. -->

The proposal has two parts:

1. Designing a new calling convention for Wasm that supports tail calls.
2. Designing trampolines for Wasmtime that support calling into and out from
   this new Wasm calling convention.

Note that we *must* do (1) because the native calling conventions don't support
general tail calls and that means that we *must* do (2) in turn since we will
need trampolines to, at minimum, convert between different calling
conventions. Given that we are doing both, however, the design of each is
largely orthogonal.

## New CLIF Instructions
[proposal-instructions]: #new-clif-instructions

First, we will introduce two new CLIF instructions: `return_call` and
`return_call_indirect`. These are analogous to the Wasm tail calls proposal's
instructions of the same name. These instructions are block terminators, the
caller and callee function signatures must use the new Wasm calling convention
(see below), and the caller and callee must return the same types. The
`return_call` instruction has a static callee immediate while the
`return_call_indirect` instruction takes a dynamic callee operand.

## New Wasm Calling Conventions in Cranelift
[proposal-calling-conventions]: #new-wasm-calling-conventions-in-cranelift

We will define one new calling convention for Wasm, the `wasm` calling
convention, implemented for each of our target
architectures[^per-architecture]. The main difference from the native calling
conventions will be that callees will clean up stack space for callers,
regardless whether we are in a tail call or not. This is required because we
need to clean up the stack arguments after exiting a function, and we no longer
always exit a function by returning to the caller, we can also exit the function
by tail calling to another function. In that new case, we can't delay cleaning
up stack arguments until the last function in a tail call chain returns to the
original caller. If every function in the tail call chain allocated stack
arguments, this would lead to *O(n)* stack growth in the tail call chain, but we
*must* maintain *O(1)* stack space for the whole chain. So callees must be
responsible for cleaning up their own stack arguments.

[^per-architecture]: Yes, one calling convention per-architecture not one
    per-target. Currently Wasm compiled for x86-64 has a different calling
    convention depending on if we are targeting Windows or Unix. That was
    motivated by having the Wasm calling convention interoperable with the
    host's native calling convention. Given that we have to support tail calls,
    we can't have that interoperability anymore, so we can also switch to a
    single calling convention on x86-64. Yay, fewer variants to mantain!

The other tricky case is when we have two function that are mutually recursive,
and one of them has more stack arguments than the other:

```rust
fn f(...register_args, stack_arg_0) {
    let x = /* ... */;
    let y = /* ... */;
    return_call g(...register_args, x, y);
}

fn g(...register_args, stack_arg_0, stack_arg_1) {
    let z = /* ... */;
    return_call f(...register_args, z);
}
```

We must either shrink the stack argument space when calling `f` from `g` even
though we have more than enough capacity for `f`'s one stack argument, or we
must dynamically reach a steady state of capacity for two stack arguments when
calling `g` from `f`. If we don't, and we always adjust the existing stack
argument capacity up when a tail callee needs more stack arguments than its tail
caller, we can get into unbounded stack-growth failure modes like the following:

* `f` calls `g`, and `g` needs more stack arguments than `f`, so it grows the
  capacity from 1 to 2
* `g` calls `f`, reusing existing stack argument capacity
* `f` calls `g`, and `g` needs more stack arguments than `f`, so it grows the
  capacity from 2 to 3
* `g` calls `f`, reusing existing stack argument capacity
* `f` calls `g`, and `g` needs more stack arguments than `f`, so it grows the
  capacity from 2 to 3
* etc...

The proposed design avoids dynamically tracking stack argument capacity by
shrinking stack argument capacity on tail calls. This does mean that we may need
to copy the return address to a new location on the stack when shrinking
capacity, but it does let us avoid dynamic capacity checks.

With that out of the way, lets get into the details.

The new Wasm calling conventions will use the frame layout in the following
diagram. For exposition's sake, this diagram assumes that the pointer size is 64
bits and that every stack argument and stack slot is pointer-sized. The actual
implementation will not (for example) use a 64-bit slot for a 32-bit argument;
it will use a 32-bit slot (and the next stack argument will pad to its type's
natural alignment if necessary). The diagram is just easier to read this way.

```
                    |       |                     |
                    |       |                     |
  |                 |       |                     |
  |                 | n/a   | Caller Frame        |
  |                 |-------|---------------------|
stack            /  | ...   | ...                 |
  |             /   | FP+32 | Stack Argument 1    |
grows           |   | FP+16 | Stack Argument 0    |
  |             |   |-------|---------------------|
down     Callee |   | FP+8  | Return Address      |
  |      Frame  |   |-------|---------------------|
  |             |   | FP+0  | Prev. Frame Pointer | <--- FP
  |             |   |-------|---------------------|
  |             |   | ...   | ...                 |
  V             \   | SP+8  | Callee Stack Slot 1 |
                 \  | SP+0  | Callee Stack Slot 0 | <--- SP
                    +-------+---------------------+

```

That is, a frame is composed of the following components:

* First are the stack arguments, if the callee function takes more arguments
  than fit into registers.
* Next is the return address, pushed by the `call`, at `FP+8`.
* Next, at `FP+0`, is the previous frame's frame pointer.
* The callee's local stack slots (including any multiple-return-value spaces for
  non-tail calls it makes) follow after that.
* Finally, if necessary, padding is inserted to keep `SP` aligned to whatever
  the architecture requires. (This is not depicted in the diagram, to simplify
  the example `SP`-based addressing.)

The crucial difference from native calling conventions is that the callee is
required to clean up the stack arguments upon its exit (whether via return or
tail call). Additionally, because we still maintain frame pointer chains, we
continue to support our fast stack walker, as well as any frame pointer-based
stack walking tools that exist externally to Wasmtime (debuggers, profilers,
etc...).

The following pseudocode for function prologues, epilogues, calls, and tail
calls shows how we will maintain this frame layout:

```rust
fn call() {
    // When beginning a call sequence, the caller's frame is on top of the
    // stack.
    //
    //   FP               SP
    //   |                |
    //   V                V
    // [ ...caller-frame... ]

    // When computing arguments and placing them in their locations as specified
    // by the calling convention, the caller will push stack arguments,
    // incrementing `SP`.
    compute_args_and_move_them_to_registers_and_stack_slots();

    //   FP                                  SP
    //   |                                   |
    //   V                                   V
    // [ ...caller-frame... | ...stack-args... ]

    // Next we transfer control to the callee, pushing our return address (RA)
    // to the stack just before jumping to the callee.
    //
    //   FP                                      SP
    //   |                                       |
    //   V                                       V
    // [ ...caller-frame... | ...stack-args... | RA ]
    call $callee;

    // When the call returns, the stack arguments and return address have been
    // popped off the stack for us by the callee (see the epilogue for details).
    //
    //   FP               SP
    //   |                |
    //   V                V
    // [ ...caller-frame... ]
}

fn prologue() {
    // Upon entry to the function, the previous frame's FP is still active and
    // the SP points to our return address.
    //
    //   FP                                      SP
    //   |                                       |
    //   V                                       V
    // [ ...caller-frame... | ...stack-args... | RA ]

    // Save the caller's frame pointer onto the stack.
    push FP;

    //   FP                                           SP
    //   |                                            |
    //   V                                            V
    // [ ...caller-frame... | ...stack-args... | RA | FP' ]

    // The callee's frame pointer is the current stack pointer.
    FP = SP;

    //                                              FP/SP
    //                                                |
    //                                                V
    // [ ...caller-frame... | ...stack-args... | RA | FP' ]

    // Allocate the callee's stack frame by decrementing SP.
    SP = SP - size_of_callee_frame();

    // The callee's frame is now initialized.
    //
    //                                                FP                    SP
    //                                                |                     |
    //                                                V                     V
    // [ ...caller-frame... | ...stack-args... | RA | FP' | ...stack-slots... ]
}

fn epilogue() {
    // When returning from the function, the stack has the following layout:
    //
    //                                                FP                    SP
    //                                                |                     |
    //                                                V                     V
    // [ ...caller-frame... | ...stack-args... | RA | FP' | ...stack-slots... ]

    // Deallocate the stack slots.
    SP = FP;

    //                                              SP/FP
    //                                                |
    //                                                V
    // [ ...caller-frame... | ...stack-args... | RA | FP' ]

    // Restore the previous frame's FP.
    FP = pop;

    //   FP                                      SP
    //   |                                       |
    //   V                                       V
    // [ ...caller-frame... | ...stack-args... | RA ]

    // Transfer control back to the caller, popping the return address and stack
    // arguments.
    //
    // On x86_64, this can use the `ret <imm16>` form that pops an immediate
    // number of bytes from the stack after popping the return address.
    //
    // Our other architectures store the return address in a register, and so
    // they can load the return address and increment the SP as necessary to pop
    // the stack arguments. Aarch64 can even fuse these two operations, IIUC,
    // unsure about others.
    return_and_pop_stack_args();

    // The caller's frame has been restored.
    //
    //   FP               SP
    //   |                |
    //   V                V
    // [ ...caller-frame... ]
}

fn tail_call() {
    // When tail-calling out of the function, the stack initially has the
    // following layout:
    //
    //                                                FP                    SP
    //                                                |                     |
    //                                                V                     V
    // [ ...caller-frame... | ...stack-args... | RA | FP' | ...stack-slots... ]

    // Note that putting the new stack arguments into their slots can involve
    // shuffling values around within the stack frame. This is similar to the
    // "parallel-copy problem" in SSA at join points. More details on this
    // later.
    //
    // Additionally, in the case where our current stack argument capacity does
    // not match the callee's stack argument capacity, we have to save temporary
    // copies of our caller's frame pointer and return address, and restore them
    // into the correct stack location after growing or shrinking the stack
    // argument capacity.
    //
    // After the new stack arguments are in place, the rest of the old
    // stack arguments and old stack slots are now garbage.
    if const { current_stack_args != callee_stack_args } {
        // Save the old FP and return address in temporary registers.
        let old_fp = load FP+0;
        let ret_addr = load FP+8;
        // Shuffle the stack arguments into their correct places.
        compute_args_and_move_them_to_registers_and_stack_slots();
        // Adjust FP by the stack argument capacity delta, to preserve the exact
        // capacity required by the callee.
        FP = FP - const { current_stack_args - callee_stack_args };
        // Restore the old frame pointer and return address.
        store FP+8 = ret_addr;
        store FP+0 = old_fp;
    } else {
        compute_args_and_move_them_to_registers_and_stack_slots();
    }


    //                                                    FP                SP
    //                                                    |                 |
    //                                                    V                 V
    // [ ...caller-frame... | ...new-stack-args... | RA | FP' | ...garbage... ]

    // Deallocate the current stack frame's garbage stack slots.
    SP = FP;

    //                                                  SP/FP
    //                                                    |
    //                                                    V
    // [ ...caller-frame... | ...new-stack-args... | RA | FP' ]

    // Restore the previous frame's FP.
    FP = pop;

    //   FP                                          SP
    //   |                                           |
    //   V                                           V
    // [ ...caller-frame... | ...new-stack-args... | RA ]

    // Transfer control to the callee. Do not use a call that pushes a return
    // address, that is already filled in. Just use a plain unconditional jump.
    jump $callee;
}
```

Under this scheme, regular call paths should be exactly as performant as they
are today. Tail calls will be as well, if they do not use stack arguments, or
the tail caller and tail callee use the same number of stack arguments. Tail
calls where there are different numbers of stack arguments will be penalized,
however. Potentially fairly egregiously, since copying return addresses around
on the stack can negatively interact with the processor's speculation.

Now let's explore shuffling stack arguments within a frame during a tail
call. Consider the following functions:

```rust
fn f(...register_args, stack_arg_0, stack_arg_1) {
    return_call g(..., stack_arg_1, stack_arg_0);
}

fn g(...register_args, stack_arg_0, stack_arg_1) {
    // elided...
}
```

When `f` tail calls to `g`, its stack arguments need to be swapped: `f`'s first
stack argument becomes `g`'s second stack argument and `f`'s second stack
argument becomes `g`'s first stack argument. If we naively copy `stack_arg_1`
from its stack argument slot in `f` to the stack argument slot that `g` wants it
to be in, then we overwrite `stack_arg_0` which we will need to fill into `g`'s
second stack argument after this. Swapping `stack_arg_0` and `stack_arg_1` needs
to happen all at once. This is similar to the "parallel-copy problem" in SSA
when implementing phis for control flow joins. We can initially start with a
simple implementation that allocates more stack space for temporaries than
necessary, and eventually replace it with a more-optimized version that keeps
the number of copies and additional space overhead to a minimum. It's a pretty
well-explored space, we just need to invest the engineering time once it becomes
a priority to improve. Perhaps we can even reuse or copy-and-adapt `regalloc2`'s
implementation.

You might also notice that, under the proposed design, tail calls essentially
perform all the same steps that function epilogues do, and then the callee
prologue resets basically the same state that the tail call just reset. The tail
call restores the previous frame's `FP` and then the prologue saves it again. We
are very much following the "return and then call" semantics of a tail call,
rather than "just" emitting a jump. The only thing that changes between the tail
caller and the tail callee frames (other than the contents of the stack
arguments and slots) is precisely how much the `SP` was adjusted past the
previous frame. As a future optimization, we could shave away this
restore-and-then-save again overhead if every function had two entry points: one
for regular calls, that has a prologue to set up the callee frame, and another
entry point just after the prologue for tail calls that assumes the frame is
already established. Then, after computing arguments and placing them in the
correct locations, tail callers would simply adjust `SP` to the callee frame's
specifications and jump to this second function entry point. There would be no
mucking around with restoring the previous frame's `FP` and then immediately
saving it again.

Another question you may have is: do we have to do anything special with
multiple-return spaces? The answer is no, we don't have to do anything special,
and a tail caller just forwards the pointer it receives along to its tail
callee. This is because the tail callee must have the same return type as the
tail caller and so must also be expecting to be passed a pointer to a
multiple-returns space of the same size as was given to the tail callee. Tail
callers never have to allocate a multiple-return space in their frame.

All architectures will use this proposed calling convention and frame
layout. The decisions we will need to make on a per-architecture basis will
basically come down to which architecture-specific registers are used for
passing arguments, which are for returns, and which are callee vs caller
saved. This is a bike shed I'd rather not paint in this RFC. We can get into the
details when implementing the calling conventions. I'm inclined to just copy the
architecture's predominant calling convention's register assignments unless we
have strong motivation otherwise. No need to reinvent the wheel just because we
are replacing our bicycle with a tricycle.

##  New Trampolines and `VMCallerCheckedAnyfunc` Changes
[proposal-trampolines]: #new-trampolines-and-vmcallercheckedanyfunc-changes

There are two primary goals for the new trampolines, beyond just supporting
correct tail calls functionality:

1. Have the minimum necessary number of trampolines for a given call: zero if
   the calling conventions match and the boundary between the host and Wasm is
   not being crossed, otherwise a single trampoline. Never chain trampolines
   together.
2. Remove all hand-written assembly. The trampolines should be generated by
   Cranelift or monomorphized Rust.

We will accomplish (1) by switching from a single, common calling convention for
all functions to agree upon (and chaining additional trampolines when a caller
or callee doesn't actually speak that calling convention) to letting callees
provide a function pointer for each of our calling conventions. One of the
function pointers will be the actual callee, while the two others will be
trampolines that convert between the different calling conventions and do any
additional work needed (such as save registers when crossing into our out from
Wasm). Callers simply choose the one that matches their calling convention and
call it. In practice, this means we will remove the
`VMCallerCheckedAnyfunc::func_ptr` field and replace it with three new function
pointer fields: `wasm_call`, `native_call`, and `array_call`.

As in the current implementation, we don't actually have a different code path
for every pair in the `caller ⨯ callee` cross product, but it is nevertheless
informative to evaluate what happens in the proposed design for each of these
cases. Let's take a look and draw our diagrams again.

### `wasm → wasm`

The Wasm caller calls the callee directly (a direct Wasm call in the same
module) or indirectly (through the `call_indirect` instruction or via a Wasm
function import). Indirect calls get the callee's function pointer via the
`VMCallerCheckedAnyfunc::wasm_call` field. Since the callee is a Wasm function,
the `wasm_call` field is a pointer right to the callee, not a trampoline.

Still a happy path! This is important because these are the hottest call paths.

```
+------+                +------+
|      |---wasm-call--->|      |
| Wasm |                | Wasm |
|      |<--wasm-return--|      |
+------+                +------+
```

### `wasm → native`

Calling from Wasm out to a `wasmtime::Func::wrap` host function. We need a
trampoline because these are two different calling conventions and we need to
save exit registers for fast stack walking. The Wasm caller gets the callee via
the `VMCallerCheckedAnyfunc::wasm_call` function pointer, which is a trampoline
that does the calling convention conversion and saving of registers.

This trampoline is generated by the Wasm compiler and stored in the compiled
Wasm module. It is fine to pair a trampoline from the compiled Wasm with a host
function in this situation because we know that we have Wasm loaded in the
system since it is a Wasm caller.

```
+------+                +------------+                  +------+
|      |---wasm-call--->|            |---native-call--->|      |
| Wasm |                | Trampoline |                  | Host |
|      |<--wasm-return--|            |<--native-return--|      |
+------+                +------------+                  +------+
```

Note: Libcalls will also use these trampolines and call path, even though we
never fully materialize them as a `VMCallerCheckedAnyfunc`. We just have to
ensure that we also generate a copy of these trampolines for each libcall (or
libcall signature if we are deduplicating trampolines) used in a Wasm module
that we are compiling.

### `wasm → array`

Calling from Wasm out to a `wasmtime::Func::new` host function. We need a
trampoline because these are two different calling conventions and we need to
save exit registers for fast stack walking. The Wasm caller gets the callee via
the `VMCallerCheckedAnyfunc::wasm_call` function pointer, which is a trampoline
that does the calling convention conversion and saving of registers.

This trampoline is generated by the Wasm compiler and stored in the compiled
Wasm module. It is fine to pair a trampoline from the compiled Wasm with a host
function in this situation because we know that we have Wasm loaded in the
system since it is a Wasm caller.

This call path now involves only one trampoline when it used to have two.

```
+------+                +------------+                 +------+
|      |---wasm-call--->|            |---array-call--->|      |
| Wasm |                | Trampoline |                 | Host |
|      |<--wasm-return--|            |<--array-return--|      |
+------+                +------------+                 +------+
```

### `native → wasm`

Calling into Wasm from a `wasmtime::TypedFunc::call` caller on the host. We need
a trampoline because there are two different calling conventions and we need to
save Wasm entry registers for fast stack walking. The caller gets the Wasm
function pointer via the `VMCallerCheckedAnyfunc::native_call` function pointer,
which is a trampoline that does the calling convention conversion and the saving
of registers.

This trampoline is generated by the Wasm compiler and stored in the compiled
Wasm module. It is fine to pair a trampoline from the compiled Wasm with a host
function in this situation because we know that we have Wasm loaded in the
system since it is a Wasm caller.

```
+------+                  +------------+                +------+
|      |---native-call--->|            |---wasm-call--->|      |
| Host |                  | Trampoline |                | Wasm |
|      |<--native-return--|            |<--wasm-return--|      |
+------+                  +------------+                +------+
```

### `native → native`

The caller is a `wasmtime::TypedFunc::call` invocation and the callee is a host
function defined via `wasmtime::Func::wrap`. This doesn't actually cross the
boundary into Wasm code, and the calling conventions are the same, so no
trampolines are necessary. The host caller gets the callee via the
`VMCallerCheckedAnyfunc::native_call` function pointer, which is the callee
itself.

This is now a happy path with zero trampolines, when it used to have two
trampolines!

```
+------+                  +------+
|      |---native-call--->|      |
| Host |                  | Host |
|      |<--native-return--|      |
+------+                  +------+
```

### `native → array`

The caller is a `wasmtime::TypedFunc::call` invocation and the callee is a host
function defined via `wasmtime::Func::new`. This doesn't cross the boundary into
Wasm code, so it doesn't need to save registers for stack walking, but it does
need a trampoline to convert between the calling conventions. The host caller
gets the callee via the `VMCallerCheckedAnyfunc::native_call` function pointer,
which is a trampoline that does the calling convention conversion.

This trampoline will be a monomorphized Rust function defined inside the
`macro_rules!` that defines our `IntoFunc` implementations. Note that it cannot
be in any compiled Wasm object because there is no guarantee that there is any
Wasm loaded in the system (let alone with the right function signature).

This call path now involves only one trampoline when it used to have three!

```
+------+                  +------------+                 +------+
|      |---native-call--->|            |---array-call--->|      |
| Host |                  | Trampoline |                 | Host |
|      |<--native-return--|            |<--array-return--|      |
+------+                  +------------+                 +------+
```

### `array → wasm`

The caller is a `wasmtime::Func::call` invocation and the callee is a Wasm
function. This calls into Wasm from the host so we need a trampoline to save
entry registers for stack walking and to convert between the different calling
conventions. The Wasm caller gets the callee via the
`VMCallerCheckedAnyfunc::array_call` function pointer, which is a trampoline
that does the calling convention conversion and saving of registers.

This trampoline is generated by the Wasm compiler and stored in the compiled
Wasm module. It is fine to pair a trampoline from the compiled Wasm with a host
function caller in this situation because we know that we have Wasm loaded in
the system since it is a Wasm callee.

This call path used to involve two trampolines, but now has only one!

```
+------+                 +------------+                +------+
|      |---array-call--->|            |---wasm-call--->|      |
| Host |                 | Trampoline |                | Wasm |
|      |<--array-return--|            |<--wasm-return--|      |
+------+                 +------------+                +------+
```

### `array → native`

The caller is a `wasmtime::Func::call` invocation and the callee is a
`wasmtime::Func::wrap` host function. This doesn't cross into Wasm, so it
doesn't need to save registers, but we do need a trampoline to convert between
the different calling conventions. The host caller gets the callee function
pointer from the `VMCallerCheckedAnyfunc::array_call` field, which is the
trampoline's function pointer.

This trampoline will be a monomorphized Rust function defined inside the
`macro_rules!` that defines our `IntoFunc` implementations. Note that it cannot
be in any compiled Wasm object because there is no guarantee that there is any
Wasm loaded in the system (let alone with the right function signature).

This call path used to involve two trampolines, but now has only one!

```
+------+                 +------------+                  +------+
|      |---array-call--->|            |---native-call--->|      |
| Host |                 | Trampoline |                  | Host |
|      |<--array-return--|            |<--native-return--|      |
+------+                 +------------+                  +------+
```

### `array → array`

The caller is a `wasmtime::Func::call` invocation and the callee is a
`wasmtime::Func::new` host function. This doesn't cross into Wasm, so it doesn't
need to save registers for stack walking, and the calling conventions match so
it doesn't need any trampolines. The caller gets the callee function pointer
from `VMCallerCheckedAnyfunc::array_call` which is the callee's actual function
pointer, not a trampoline.

This call path used to involve four trampolines, but now has none!

```
+------+                 +------+
|      |---array-call--->|      |
| Host |                 | Host |
|      |<--array-return--|      |
+------+                 +------+
```

### Nullability of `VMCallerCheckedAnyfunc::wasm_call`

You may have noticed that the `VMCallerCheckedAnyfunc::wasm_call` function
pointer always points to a function or trampoline that is defined inside a
compiled Wasm module. What if there is no Wasm loaded in the system?

The field is actually nullable, and will be `null` if there is no Wasm loaded in
the system. This nullability is fine, and we don't ever need to dynamically
check for it in our call paths, because the field will never be used except for
by a caller that is speaking the Wasm calling convention. The only caller that
speaks the Wasm calling convention is Wasm itself, and that means that Wasm is
loaded in the system and therefore the field will not be null. We just have to
maintain this invariant, and possibly fill in the null field, when passing
`VMCallerCheckedAnyfunc`s into the Wasm's `vmctx` at instantiation time. And we
will have the appropriate trampoline available to fill in that field when
instantiating the Wasm because the trampoline will be inside the compiled Wasm
module. It works out perfectly!

## Incremental Implementation Plan
[incremental-plan]: #incremental-implementation-plan

Here is a rough check list of work needed to implement tail calls. Of course,
plans often change when we get into the nitty gritty details, but this should be
a reasonable set of incremental milestones we can work towards.

* [ ] Add support for the tail calls proposal in `wasm-tools`:
  * [X] `wat` and `wast`
  * [X] `wasmparser`
  * [ ] `wasm-encoder`
  * [ ] `wasm-smith`
* [ ] Add `return_call` and `return_call_indirect` instructions to Cranelift
      (just instructions and validation; lowerings unimplemented)
* [ ] Implement new calling conventions in Cranelift and implement lowerings for
      `return_call[_indirect]` instructions
* [ ] Update `VMCallerCheckedAnyfunc` and Wasmtime's trampolines to use multiple
      function pointers instead of chaining trampolines
* [ ] Add support for tail calls in Wasmtime (which should just be switching to
      the new Wasm calling convention and tying a bow on all the previous work
      at this point)

# Rationale and Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- What other designs have been considered and what is the rationale for choosing -->
<!-- the one proposed? -->

This proposal is not perfect, without any trade offs. There are a couple
downsides associated with the proposed trampoline design:

* It does introduce some new memory overhead, since we grow the size of
  `VMCallerCheckedAnyfunc` from three words to five words. This is a fundamental
  space-time trade off, where our current design (unwittingly) trades time for
  space by chaining together trampolines instead of keeping around multiple
  function pointers, each for use by a different kind of caller. I believe that
  this space overhead is overall acceptable since `VMCallerCheckedAnyfunc`s
  account for a small fraction of the total space needed for a Wasm instance,
  and not even all Wasm functions need to be materialized as
  `VMCallerCheckedAnyfunc`s in the first place.

* Multiple copies of identical trampolines across different compiled Wasm
  modules are not deduplicated. That deduplication is not feasible unless we
  give up AOT compilation, and instead JIT compile trampolines the first time we
  actually need them and haven't already JIT compiled that particular
  trampoline. This seems like an unavoidable cost given that relying on JIT
  compilation is a nonstarter.

That said, it still puts us in a better place than any of the alternative
trampoline designs:

* Require a compiler to always be built into Wasmtime so we can JIT trampolines
  on demand to make certain function calls. But requiring JIT compilation is a
  non-starter.

* Dynamically detect when we would need a trampoline but don't have one (i.e. we
  would need JIT compilation) and panic in these scenarios. Even if rare, this
  is very opaque and hard to understand for Wasmtime embedders unless they know
  the details of Wasmtime's implementation.

* Dynamically inspect the callee and caller pair, and branch to different code
  paths depending on whether it is Wasm that needs a trampoline or is a
  host-to-host call that doesn't need trampolines. This adds complexity and
  overhead to our call paths. We want these to be as maintainable as possible
  and definitely as fast as possible.

* Add Cranelift as a build dependency for Wasmtime and pre-compile a trampoline
  for each of our `IntoFunc` signatures so we always have trampolines available
  for host functions. In addition to complicating the build process, this bloats
  the binary with a bunch of trampolines that are probably never used, which
  makes it harder to use Wasmtime in resource constrained environments.

* Use the type system to differentiate between `wasmtime::HostFunc` and
  `wasmtime::WasmFunc` and then statically disallow calls that would need some
  sort of trampoline that we wouldn't have access to. This is a large, clunky
  change to Wasmtime's API for very little benefit, especially since we have
  viable alternatives.

As far as the new Wasm calling convention goes, there are some alternative
approaches that we could explore as well:

* We could bend over backwards to maintain interoperability with existing
  calling conventions (to a degree) the way that [SpiderMonkey plans on
  doing](https://docs.google.com/document/d/19Q5yMe7GUwi8T7ynWtCcmmd1DPk9xz2GQVOInh_NR3U/edit).
  This involves making the first tail call a regular call and then only
  subsequent tail calls in the chain reuse stack frames. This is subtle and
  requires dynamically checking whether a tail call is the first or not, which
  our proposed design avoids. It does make sense for SpiderMonkey, though, where
  there are more tiers of compilers that all need to agree with each other and
  with the host, and there are more kinds of pinned registers and context to
  switch, save, and restore between different types of calls. They have a lot
  more inertia than we do.

* We could do `push FP` in callers, not the callee prologue. This would let us
  implicitly compute the stack argument space's current capacity by comparing
  `FP` and `SP` without requiring additional bookkeeping. Then we can grow the
  stack argument space as necessary when tail calling a function that needs more
  capacity than we currently dynamically have, until we reach a steady state of
  capacity. In addition to introducing dynamic checks in tail calls compared to
  the proposed calling convention, this means that we swap the order of the
  saved frame pointer and return address on the stack compared to their usual
  ordering. This would break any external tooling (debuggers, profilers, etc...)
  that would otherwise be able to walk our stacks. It also is a little bit of a
  code size pessimization since it moves a little more work into *O(n)* callers
  from a single callee.

# Open questions
[open-questions]: #open-questions

<!-- - What parts of the design do you expect to resolve through the RFC process -->
<!--   before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through implementation after -->
<!--   the RFC is accepted? -->

* Should we deduplicate all of our trampolines (within a compiled Wasm module)
  on a per-signature basis? This saves disk and memory space but means that
  trampolines will have a slower indirect call to the callee instead of a faster
  direct call.

  The alternative is to have a unique trampoline per function that can escape
  (i.e. is exported, can go in an exported table, or be turned into a `funcref`
  and passed to the host). Because the trampoline has a unique callee, it can do
  a direct call. However, more trampolines mean more disk and memory usage.

  Note that we could make different decisions for different kinds of
  trampolines, based on their expected usage frequency and how hot we expect
  them to be. For example, we might want to de-duplicate trampolines for `wasm ⟷
  array` calling conventions (generally only used for slow paths) but have
  unique trampolines for `Wasm ⟷ native` calling conventions (very hot paths).

* What sort of interaction does the proposed calling convention have with stack
  maps, reference types, and GC? I am fairly sure we don't need to worry about
  anything as long as callee stack maps "own" stack arguments, since the caller
  can no longer exist on the stack anymore in the case of tail calls, and that
  could otherwise lead to use-after-free bugs from failing to trace some stack
  slots. I think it is good to call out this area of interaction though just to
  force some more people to think about it and hopefully come to the same
  conclusion as me.

* What is the interaction with exceptions? We don't implement Wasm exceptions
  yet, but we will want to eventually. Is the proposed calling convention
  compatible? I don't see any reason why not! But not 100% sure. Again, I just
  want to call this out to get a few more people thinking about it and double
  checking.

* Should we repurpose the `wasmtime_*` calling convention variants into the new
  Wasm calling conventions, or should we create wholly new calling conventions?
  I'm leaning towards the latter so we have less existing stuff to disentangle
  and tests to modify and keep passing as we are in the just-start-building-it
  phases. But maybe others feel differently?

* What is the interaction with Windows' SEH and the assumptions that Windows
  makes about user-space programs? Is this going to break unwinding from system
  calls on Windows? I think it can, but also I don't see a way that we can avoid
  breaking that, even if we did what SpiderMonkey is doing to provide some
  limited interoperability with the native calling conventions, when there is a
  tail call chain on the stack and the OS is trying to unwind the user-space
  program. I think this is unavoidable.

* Can we reuse, or fork, code from `regalloc2` to do parallel copies between
  stack arguments?

* Is copying the return address between stack slots on tail calls (where we have
  different numbers of stack arguments) incompatible with hardware control-flow
  integrity features? I don't believe it conflicts with ARM's pointer
  authentication scheme, but I'm less sure about Intel CET and its shadow stack.
