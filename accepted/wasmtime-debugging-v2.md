# Summary

This RFC updates our plans laid out in the [first debugging
RFC](wasmtime-debugging.md). It proposes implementing guest debugging
in Wasmtime to build on Cranelift rather than Winch, based on new
realizations about possible strategies; it describes in detail the
lowered code sequences, data structures and algorithms we plan to use.

The structure of this "update RFC" comes in three parts: problems with
the current plan-of-record that make it difficult or impossible to
satisfy our stated goals via its planned steps; a series of revisions
to the plan-of-record; and a description of mechanisms that enable
that plan-of-record to be practical.

# Motivation

## Current Plan-of-Record (First RFC)

The current plan-of-record for debugging in Wasmtime is to:

- Build the debugger on top of code execution in the Winch single-pass
  compiler tier;
- Implement breakpoints and watchpoints by making hostcalls from
  compiled Winch code into the Wasmtime runtime, potentially for every
  step-point and for every Wasm load and store.
  
This plan is motivated by implementation pragmatism: mainly, the first
RFC builds on the assumption that the Wasm virtual machine-level state
(locals and operand stack) is easy to recover from Winch's mapping to
machine state, but hard or impossible in the general case in an
optimizing compiler. As the first debugging RFC states:

> Interactive debugging support in an optimizing compiler like
> Cranelift would require fundamental changes to how optimizations are
> applied and how program state is tracked throughout
> compilation. Maintaining compiler performance while integrating
> those changes would be a significant undertaking. For this reason,
> for the foreseeable future, we are only going to support live
> debugging with Wasmtime’s baseline compiler: Winch.

## Problem: Implementation Completeness

The initial impetus for the thinking that led to this RFC was an
observation of the current state of feature support in the various
compiler backends in Wasmtime. In particular, there was a realization
that the substantial work that would be required to bring Winch up to
feature parity with Cranelift does not square with current staffing
realities, and the substantially delayed "payoff" of working debugging
could mean that it would be hard to justify the work at all. If there
were any way to avoid this and find a shorter path to a working Wasm
VM-level debugger, it would be preferable.

In particular, Winch does not support:

- Garbage collection or function references;
- Exception handling;
- Tail calls;
- Stack switching (nearly landed on Cranelift);
- SIMD on any platform except x86-64;
- Any platform other than x86-64 at a tier-1 level;
- Any platform other than x86-64/aarch64 at all.

While an initial proof-of-concept of debugging for core Wasm on Winch
would be valuable, questions would immediately arise about support for
many of these features -- and they are not niche features, either:
several high-level languages have plans to compile to Wasm using GC
and exceptions, SIMD is becoming more widely used for all sorts of
in-guest compute kernels and library functions, tail calls will be
non-negotiable for some guest languages and an important optimization
for others (e.g., interpreter loops), stack-switching is likely to
become important for guest green-threading implementations such as
Go's goroutines, and platforms other than x86-64 and aarch64, while
less widely used, are important for Wasmtime's portability story.

In a world with infinite engineering time, we would have
implementation parity eventually across all tiers, and it is possible
that support for new proposals and features might even come to the
baseline tier *first* someday (as already happens in mature browser
engines such as SpiderMonkey). However, our reality today is one of
being "stretched extremely thin", and duplicating all of the
substantial work for any of the above to *also* support it in Winch,
to allow such modules to be debugged, is daunting and hard to
justify. (At least, for the author in the present employment context
-- in which at least GC and SIMD will be immediately needed!)
Bringing Winch to full parity probably requires between 6 months and a
year of work, based on the effort taken in Cranelift and "downscaling"
for lower baseline-tier complexity.

None of this argument is meant to take away from the immense effort
put into Winch nor the value it provides: it is only to say that our
debugging functionality should support our full range of Wasm
proposals, not a subset.

So far we have discussed why there are significant costs to taking the
currently-planned approach based on Winch, but we have not yet
addressed whether the original arguments against Cranelift -- need for
"fundamental changes" and daunting complexity -- still hold. We now
describe why they may not.

## Realization: Debugging State as an Explicit Output

The original debugging RFC based its estimation of effort for a
Cranelift-based solution on the assumption that we would have
continued the *conventional approach* of augmenting a compiler
pipeline with debug metadata: producing, lowering, and emitting
descriptions of where to find each bit of state from the input program
in the output machine code. In other words, the assumption is that we
would have to produce a precise map of Wasm locals and the operand
stack, at each Wasm opcode point, in terms of machine registers,
spillslots, and/or other on-demand computations. In fact, Cranelift
today has infrastructure for this, and the Wasm-to-CLIF translator
embeds metadata in the CLIF ("value labels") that are taken through
regalloc to produce such maps. The conclusion from our experience with
this system is that it is brittle at best, and trying to make it fully
precise in all cases would be an enormous uphill battle. What's more,
every new optimization, especially any that perform code motion, would
need to be designed against this requirement.

However, recently the author came to a new realization: one can see
this "debug view" of the Wasm virtual machine state as *another
output* of the computation, and can produce CLIF that generates this
view, interspersed with the rest of the computation. In other words,
we create a stackslot (region on the stack) sized to contain all Wasm
locals and the maximum operand stack depth; then store values to this
region as they are produced or updated. (The operand stack depth at
any program point is statically known and can be included in
metadata.)[^1]

[^1]: One concrete inspiration for this approach was the "dynamic
      context" mechanism added to support Wasm exceptions: rather than
      struggle to reverse-engineer some bit of state (there, the
      dynamic instance identity of a given frame), we took a generic,
      low-level approach by adding a "context value" item to exception
      tables and passing the value through. In other words, if the
      runtime needs some state, we should ("simply") change the
      compiled code to store that state.

Because these stores are side-effects, from Cranelift's point of view,
they will not be moved or altered in any way. Every compiler
optimization is obliged to maintain their same order and the same
stored values. The original Wasm VM-level machine state view is
carried through from Wasm translation to the execution behavior of the
machine code.

The runtime performance of code instrumented in this way will
certainly be lower than that of Cranelift-optimized code; but
un-altered performance has never been the goal in a "debug mode" (see:
the plan to use Winch instead). In fact the performance should be at
least on par with Winch -- which performs almost all Wasm state
updates as stores to memory (local slots or the actual machine stack),
modulo some block-local register allocation. The Cranelift-based
approach keeps those stores, but avoids the loads (generally; see
below). Other optimizations could still apply: for example, Cranelift
can still simplify the CLIF generated by garbage-collection operator
lowerings, or memory bounds-checking.

One can see this as a kind of factoring into orthogonal concerns, with
Cranelift itself remaining a low-level "core calculus" that describes
the *implementation* of other concerns. This is a theme we have
followed in other domains in the Wasmtime/Cranelift interaction
too. For example, we [moved stack-map logic outside of core
Cranelift](https://bytecodealliance.org/articles/new-stack-maps-for-wasmtime),
such that the slots seen by the runtime at safepoints are "just opaque
stores" from Cranelift's point of view.  In general, using Cranelift
as a low-level code generator and describing what we want -- state
updated in memory at certain points -- with explicit IR seems to be a
robust, flexible, and simple approach.

A concrete example of instrumentation, metadata format, and some
future design considerations will be discussed below.

# Updated Plan: Debugging on Cranelift-Compiled Code

This RFC, concretely, proposes one RFC-level decision that requires a
new consensus: to update our intended approach to debugging support in
Wasmtime to start from Cranelift-compiled code, with instrumentation
to implement the debug state and other mechanisms.

These mechanisms are sketched below, but some details may change in
small-to-medium ways as we experiment with a real implementation and
work through all of their interactions. The sketches are provided to
substantiate the new direction's feasibility; the goal of this RFC is
only to build consensus on the one most-significant bit.

# Mechansms

## Wasm virtual machine state as an explicit side-effect

We will add a new compilation configuration option to the Wasm-to-CLIF
translator (similar to fuel or epoch support) that enables "precise
debugging state"; this will be a prerequisite for the use of the
debugger interface.

### VM state

The option will add (i) a single stackslot (of statically known but
variable size) to each compiled Wasm function, holding its locals and
operand stack values; and (ii) a mechanism for referring to this
stackslot as part of the CLIF preamble that will describe its stack
location in a compilation metadata result.

For example, we might translate

```
(func (param i32 i32) (result i32)
  (i32.add (local.get 0) (local.get 1)))
```

as

```
function u0:0(i64 vmctx, i64, i32, i32) -> i32 {
  ;; Four ValRaw slots (two locals, max stack depth of two).
  ;; The `source_state` flag places its location in metadata.
  ss0 = stack_slot source_state 64
  
block0(v0: i64, v1: i64, v2: i32, v3: i32):
  store.i32 v2, ss0+0  ;; local 0 is param 0
  store.i32 v3, ss0+16 ;; local 1 is param 1
  store.i32 v2, ss0+32 ;; stack push from (local.get 0)
  store.i32 v3, ss0+48 ;; stack push from (local.get 1)
  
  ;; Step/breakpoint instrumentation here; see below.
  ;; End result is a possible call into runtime that could observe
  ;; the contents of `ss0`, and would know, via Wasmtime-level metadata,
  ;; that the operand stack has depth 2, and the types of all locals
  ;; and operand stack values.
  
  v4 = iadd.i32 v2, v3
  
  store.i32 v4, ss0+32  ;; stack push from `(i32.add ...)`
  
  ;; Step/breakpoint here too.
  
  return v4
}
```

Then, whenever control has entered the runtime and there are active
Wasm stack frames, a stack-walk can:

- Map PCs to modules and functions (note: the same would happen with a
  Winch-based approach);
- Look up Wasm-level metadata for that program point (likewise, same
  as Winch-based approach);
- Look up the location of the state stackslot relative to the start of
  the frame (this is new);
- Read the locals and operand stack values off of the stackslot (this
  is almost the same as a Winch-based approach, except at some stack
  offset rather than "current SP").
  
The overall complexity to read Wasm VM-level state is more or less the
same as in a Winch-based approach -- finding the offset of the
stackslot is new, but Winch goes to special trouble to 16-align the
machine stack, which would require some care to correct for. In either
case, once we have the stack offset, we can directly read `ValRaw`s.

### Implementation in Wasm Translator

The insertion of these store instructions should be fairly
well-factored from the implementation of each individual Wasm
operator. In particular, we already have mechanisms that track Wasm
local values, and the operand stack, as we generate SSA; the local-set
and value-push operations can be augmented to emit stores if the debug
option is set.

### Design Considerations and Future Extensions

The main design question appears to be: is the state a strict
*output*, i.e., read-only when the program is paused, or is it an
output *and input*, i.e., read-write? In other words, do we want to
support setting locals to new values when stepping through a program?

Perhaps surprisingly, either option is implementable: if we want to
support read-write state, we *load* new values (either eagerly or,
more likely, lazily) after each step/breakpoint. This, again, tracks
what Winch does (i.e., pop from stack), but written explicitly in IR.

Additionally, we could offer both modes as configuration options, if
desired: this would surface the performance tradeoff.

As a final interesting note: if we ever want to implement on-stack
replacement (OSR) for tiering, the "portal" by which state can be
transferred from one compilation in one tier to another in another
tier is most likely to be Wasm VM-level state (because that is all the
tiers will have in common). If we already have a mechanism to permit
reloading all Wasm-level state after a given step-point, then we
almost have OSR -- the additional step is to add rules about liveness
of *other* SSA values across step/OSR-points and regenerate them as
needed. This RFC will not explore the topic further, but it does
perhaps add some additional value to the generic mechanism.

### Interactions with Optimizer

It is important to note that a significant advantage of this scheme is
that the stores to update state, together with breakpoint-opcodes as
sequence points that form compiler barriers (below), *depend only on
correct semantics-preserving compilation* to get fully precise state
out of the machine code. In other words, we do not need to
reverse-engineer the compiler's output in the debugger, or carefully
limit ourselves from implementing certain kinds of optimizations or
transforms. Rather, because the compiler *must* always preserve
side-effects and their ordering, we get debug correctness for free:
updates to debug state, and step-poitns at which the state can be
observed, are first-class IR side-effects that a correct compiler will
preserve. Cranelift may even optimize the entire user program away (in
the read-only breakpoint mode at least), leaving only updates to debug
state at each step -- and that is fine, because we will still observe
debug state in the correct order with respect to the original program!

## Breakpoints

The original debugging RFC proposes to implement breakpoints as follows:

> for every instruction emitted, emit a call to a utility function
> that can check if execution should be paused

as part of a "check for breakpoint" pseudo-opcode. While feasible to
implement in Cranelift as well, the cost of a hostcall (from Winch or
Cranelift!) is quite high, involving state-saving, trampolines, and a
number of layers of Rust glue. (The original RFC also proposes
overwriting NOPs in the generated machine code with trap instructions
dynamically, but this carries several downsides, among them
cache-coherence and other code-publishing overheads, and unwanted
interference across separate instances of one module.) This RFC
instead proposes, again, an inline instrumentation-based approach.

In brief, we retain a "check for breakpoint" pseudo-opcode, lowered as
a sequence point (opaque to the compiler, inhibits moving
side-effects) and with some actual breakpoint logic as the lowering.
We could implement either an "explicit checks" breakpoint mode or a
dynamic-patching breakpoint mode:

- In the explicit-checks mode, The `vmctx` for any particular instance
  will store a pointer to a bitmap, with one bit per original Wasm
  opcode. At each "check for breakpoint" point, we will emit a load,
  mask, compare and branch. On the cold path at the branch
  destination, we perform the hostcall into the runtime that returns
  control to the debugger.
  
- In the dynamic-patching mode, we lower the check-for-breakpoint
  opcode as a NOP region large enough for patching in a call to a
  break trampoline. Critically, we do the patching on a *private copy*
  of the code segment. This should be possible to do reasonably
  cheaply because Wasmtime already must emit completely
  position-independent code (it has no support for load-time
  relocations in compiled artifact text segments); it only means
  adjusting function pointers placed into VM structures when
  initializing instances if an instance has a private copy of the
  module's code.

We plan to experiment with both, and choose the option that is as
simple as possible without imposing undue run-time or code-size bloat.

## Async Nature of Breakpoints

The original RFC did not discuss details of the handoff of control
between the debugger controller and debuggee (at breakpoints,
watchpoints, between steps, or otherwise); it is worth clarifying this
as well.

In particular, this RFC proposes allowing debugging only when the
guest is instantiated in an async store; all invocations into the
debuggee must be async. This is a result of the nature of the
"transfer back to debugger": the guest does not unwind its stack;
single-stepping looks like a ping-pong between two coroutines on two
different stacks, the debugger and the debuggee.

One alternative, where the debugger runs "inside" the trap back to the
runtime, seems much less ergonomic, and possibly limiting: for
example, it would split an event loop that processes commands from a
DAP debug connection across one part "outside" the main invocation,
and another replica "inside" a breakpoint that pumps the connection
until resuming execution.

What's more, if we eventually support mutations to the guest frames
themselves -- for example, forcing an early return from a function --
this is more difficult to implement if the frames are on the host
stack.

Finally, snapshotting for record/replay and time-travel debugging is
much easier to conceive of when the Wasm stack is its own entity (see
below).

We note that the design of the control-transfer mechanisms will need
to support at least:

- Run-till-breakpoint/watchpoint;
- Single-step;
- Run, with an asynchronous "interrupt" to return control to the
  debugger (DAP provides such a command and it would be a conspicuous
  omission if we do not support it).
  
The last option more or less necessitates that we periodically yield
from guest code to the host, and our existing async mechanism of fuel
is the most natural way to do this.

The host/debug-adapter API design is left unspecified here (as it is
in the original RFC), other than that it must support DAP
abstractions; but we expect that it will likely provide a "run"
invocation to the underlying VM that takes some mode (single-step, run
until breakpoint, run indefinitely) and returns some result (periodic
yield, hit a breakpoint/watchpoint, hit a trap) that can be provided
naturally on top of the async-chunks-of-work design proposed
here. This is also similar to the design of the KVM API, or of the
Pulley interpreter: execution runs until some "exit event", and can be
resumed later.

## Memory Watchpoints

Next, we consider *watchpoints*, which provide the ability to pause
execution and yield to the debugger when certain machine state
changes.

The original RFC proposes

> like the proposed implementation for single-stepping mentioned in
> the [previous section](#breakpoints-and-single-stepping), we can
> implement watch-points by instrumenting all store operations with a
> similar call out to a utility function

and describes but chooses not to take an alternative of a
virtual-memory-based approach, because of the associated
signal-handling complexities.

In this RFC, we propose an inline instrumentation-based approach
again, using the concept of a *shadow memory*. The idea is:

- For every Wasm memory that is watchpoint-able, we have a watchpoint
  store of equal size (1-to-1);
- Whenever we do a store (or load, for read watchpoints) on that
  memory, we first load data of the same size at the same address in
  the watchpoint store;
- If that data is nonzero, yield to debugger.

This allows byte-granularity watchpoints to be set, and loading the
same width handles all partial overlap cases correctly. (For example,
if we set a watchpoint on only one byte, but an `i32.store` overwrites
that byte, the initial load may see watchpoint flags of `0x01000000`,
`0x00010000`, `0x00000100`, or `0x00000001`; any of these are nonzero
and would trigger a yield to the debugger.)

This approach is relatively simple, allows fine-grained control and an
arbitrary number of watchpoints (as opposed to the limited number of
hardware watchpoints that native debuggers have available), and should
have significantly lower overhead than the currently-accepted plan of
a full hostcall on every store. We can also easily implement this
logic in host-side accessors, if desired.

If the 1-for-1 shadow memory approach is too expensive in memory
utilization for some use-cases, we may explore alternatives that use a
two-level scheme, where there is a dense map (fast constant-time
accesses) with a bit per larger granularity (e.g., page or cache
line), and a sparse hashmap of actual watchpoints that is checked in a
slow-path.

## Snapshots and Wasm Stacks

As part of our exploration of record/replay execution and eventual
time-travel debugging, we have considered in more detail what will be
required to implement snapshots so that the time-travel can have
"keyframes" to restore to (as described in the original RFC and
implemented in *rr* and some other tools).

While the tradeoffs in *memory* snapshotting have been discussed a
fair amount, Left undescribed so far is the *execution state*
snapshotting. If we periodically snapshot, we must assume that there
will be active Wasm frames on the stack: we cannot afford to wait for
execution to fully unwind to the top-level invocation (that may be an
unbounded amount of time). How do we capture and then restore this
state?

One may be tempted to "read off" the state in the form of Wasm
virtual-machine-level values, and then fully reconstruct stack frames
on restoration (and indeed, as discussed above, OSR would provide the
mechanisms for this), but there is a much simpler way, if we accept a
few restrictions. In particular, if we accept that

- Snapshots will be local to, and live only within the lifetime of, a
  `Store` (this is sufficient for time-travel debugging -- only the
  replay trace is portable and persistent);
- We implement memories and tables with fixed addresses (i.e.,
  virtual-memory scheme with full reserved guard region, rather than
  relocation on growth); and
- We keep the Wasm stack separate (as discussed under "async nature"
  above)
  
then we can implement stack snapshotting and restoration with a
*memcpy of the fiber stack* (!). This relies on the invariant, ensured
by the requirements above (around non-moving memories and living
within the lifetime of one store), that all live native pointers in
the machine code remain valid if we later restore to that point.

This has very desirable robustness properties, analogous to the
similar advantages of the instrumentation schemes above: it means that
we don't need to "reverse-engineer" Cranelift's frame layout in any
way, or ensure any special properties about which values are live
across time or updated when restoring a snapshot. Rather, the
Wasm-as-implemented-in-lowered-machine-code state is an opaque blob
that we memcpy out, and memcpy back in, and all continues as we had
left off.

In particular, the snapshot procedure is:

- `memcpy` from base of trampoline frame (exit SP) up to top of stack;
  - (note that we only need to copy the live part of the stack, not
    the full size)
- record the fiber state (saved SP).
  - (note that the fiber library itself saves all callee-saved
    registers to the stack before switching away, so we don't need to
    handle those separately)

And the restore procedure is the opposite:

- `memcpy` from the snapshot of the stack back onto the stack;
- resume to the fiber.

Notably, and perhaps surprisingly, all of this can actually be
implemented purely in the fiber library: so we have a general notion
of fiber snapshots (almost like setjmp/longjmp primitives on fibers).

## Record/Replay, Store Boundary, and Host API

Arjun Ramesh (@arjunr2) has implemented a [prototype of record/replay
in Wasmtime](https://github.com/bytecodealliance/wasmtime/pull/11284),
proving the viability of this component of our overall time-travel
debugging plans with remarkably low overhead (~4% execution time
increase). For completeness, we describe a few high-level aspects of
the design and interface here, and note its relation to the above
plans.

The general simplifying assumption behind the design is that we record
*an entire store*, and trace all interactions across the store
boundary so that we can replay them. 

The most challenging (from a performance point of view) kind of
interaction to trace is an update to Wasm memory from the host side:
for example, as a response to a hostcall requesting an IO read into a
guest buffer. Semantically this is straightforward: we augment the
APIs that provide slices of guest memory to the host code so that they
record which areas of memory become "dirty" (if granted mutable
access). This could become expensive if the host code is careless and
takes a slice of the entire memory, however. One aspect of our
approach that eases this cost, and makes it more likely that traces
will be compact and "precise" (recording only what is necessary to
reproduce the run), is interaction with the *Wasm component model* and
its *canonical ABI*, which specifies exactly which memory addresses a
hostcall may mutate.

The prototype linked above provides trace and replay modes via
Wasmtime's `Config`, but requires the host to initiate calls and other
interactions with the store; calls from the guest back *out* to the
host are recorded and replayed. In the final form, we plan to record
*both* directions, i.e., the host's calls into Wasm (recording the
arguments to calls), and the Wasm's calls back to the host (recording
the return values from calls). We will then provide two new
abstractions at the host API level: a `Trace`, which is analogous to a
`Module` as a static entity that can be executed, and a `Replay`,
which is analogous to an `Instance` as one dynamic execution of a
`Trace`. The main difference is that the `Replay` will fully own its
`Store`, rather than simply being instantiated with an existing store,
because it must fully control all actions within the store to ensure
determinism.

Once we have this infrastructure, we can provide debugging APIs on top
of it the same way we will for live instances. Note that the two
implementation efforts can proceed largely independently until we
combine them. In particular, also, note that *the recording need not
run the same machine code as the replay*: this is critical, because it
means that we can keep the same fast recording overhead in production,
and pay the cost of debugging instrumentation only when examining a
trace of a bug.

# Alternatives

We could choose not to take this new direction, and remain with the
original plan as proposed in the accepted original RFC. The tradeoffs
therein are the main subject of this RFC.

