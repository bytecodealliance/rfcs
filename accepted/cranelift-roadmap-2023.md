# Cranelift Roadmap for 2023

Following the tradition of start-of-year roadmaps (for
[2021](https://github.com/bytecodealliance/rfcs/pull/8) and
[2022](https://github.com/bytecodealliance/rfcs/pull/18)), this RFC
outlines a collection of ideas and projects that we have on the table
for 2023. The ideas range in certainty from direct, concrete tasks
that follow up on previous work and which we definitely plan to
complete; to projects that we haven't started yet, or have only
started planning, but will definitely try to complete; to some ideas
that are more speculative, but would be very interesting or valuable
if we found a way to tackle them. In the past, we have actually
achieved a good amount of our roadmap (end-of-year posts for
[2021](https://bytecodealliance.org/articles/cranelift-progress-2021)
and
[2022](https://bytecodealliance.org/articles/cranelift-progress-2022)),
and while we hope to be similarly ambitious this year, there are no
guarantees! This document instead should be seen as a collective
braindump of our beliefs regarding productive directions for future
work.

Following work in 2022, we have largely completed "core refactorings"
of the compiler: we have our new register allocator, backend, and
mid-end frameworks in place, and we have taken on quite a few
mid-sized projects throughout the stack to achieve some very nice
speedups and correctness improvements, and fill in functionality
gaps.

Given that, the theme in 2023 should largely be to carry through the
efforts we have to the point of *polished completeness*, and fill in
many of the "nice-to-have" gaps, like up-to-date and thorough
documentation and examples. There are also some good opportunities for
in-depth projects on new features that we are currently missing, but
these are likely to be more scoped and less cross-cutting than the
"core refactorings" above. In other words, the compiler is maturing
(and this is a good thing!); let's work toward furthering that!

## Compiler Performance: regalloc2 and SSA

The first compiler performance-related project we plan to tackle,
early in 2023, is the continuation of our
[regalloc2](https://github.com/bytecodealliance/regalloc2)
compile-time optimization work leveraging
[SSA](https://en.wikipedia.org/wiki/Static_single-assignment_form)
invariants at the input to the register allocator.

The last phase was completed mostly by a long-running series of
changes to eliminate all non-SSA input (multiple definitions of one
virtual register, or use of special "pinned" virtual registers that
correspond to fixed physical registers), which we have been [checking
in debug
builds](https://github.com/bytecodealliance/wasmtime/pull/5354) and
fuzzing with now for almost two months.

Now that the input to the allocator is pure SSA, we can greatly
simplify its design, unlocking potentially large compile-time
improvements. In brief, without SSA invariants, we can only keep *one
copy* of a virtual register around, because it may be mutated
throughout its lifetime. This one-copy rule complicates the allocator
in many ways: it forces unnecessary "liverange splits" and awkward
constraint rewrites. For example, when a spilled value is used, the
liverange in the spillslot ends, the value moves to a register for one
instruction, then it moves back to another liverange (in either the
same or a different spillslot). If the two spillslot allocations are
the same, and the value-use was a read (use rather than def), the
"redundant move eliminator" can eliminate the store back to the
spillslot. But this represents extra work in several ways: another
liverange to process, and the move-eliminator logic altogether. It
would be better to have a copy in the register and retain one long
spillslot allocation that we only have to process once. With SSA,
inserting moves is easier: we know that the value does not change once
defined, so we can copy from any existing liverange.

We anticipate that this could result in significant speedups: a
majority of compile-time is usually in regalloc, most of the
allocator's time is spent in the main worklist processing loop that
allocates one bundle (of liveranges) at a time, and when we compare
stats, we see that regalloc2 sometimes has 2-4x as many liveranges as
IonMonkey's allocator on the same Wasm module, due to the above
restrictions. Reducing that number should produce large swings.

## Generated-Code Quality (Regular Sprints?)

Next, we hope to perform an additional pass of generic "generated code
quality" work, driven by profiling and examination of hot code in
benchmarks.

In the past, we have periodically examined "hot blocks" and, by
reading the disassembly, seen things we could fix. Because we don't
always have a regression test for every "good" outcome, *new*
opportunities of this form sometimes appear.  When optimizing
generated code with the egraphs framework, for example, I noticed that
some changes we had made in condition (bool) representation resulted
in reification of bools rather than use of the implicit processor
flags; I fixed that
[here](https://github.com/bytecodealliance/wasmtime/pull/5391). A
"compare; set-conditional; compare; branch" sequence is obvious in
disassemblies when compared to the more standard (and expected)
"compare; branch" sequence at the end of a block, but we just hadn't
been checking regularly, and didn't have a test for this (we do now
though!).

As a lesson from that, we've observed that we should probably make
regular time for "code quality sprints": take some benchmarks, such as
those in Sightglass, profile them, and observe the hottest basic
blocks' code. If we don't see anything obviously wrong or ripe for
improvement, all the better; perhaps we can use the opportunity to
write some new regression tests for the code translations that *did*
work well. But it's likely that if we look again, after some time,
we'll find a handful of small speedups. Individually thse can be
niche, but they add up, and a mature compiler requires persistence in
this kind of work to achieve consistently good code!

## Correctness via Formal Verification

We plan to continue with our formal-verification collaboration
in 2023. This collaboration with several academics (@avanhatt,
@mpardesh, @mlfbrown, @asampson) builds off of ISLE's
formal-verification focus: by writing our compiler's rewrite rules in
a DSL, we enable automated analysis by translating the rules into
other forms (for example, SMT clauses that link semantic descriptions
of CLIF and of a machine ISA). The work will be described in more
detail elsewhere, when ready, but is already achieving some noteworthy
milestones: our collaborators' system recently was able to
[independently find a bug in a 16-bit cls
lowering](https://bytecodealliance.zulipchat.com/#narrow/stream/217117-cranelift/topic/ISLE.20verification.20found.20previously.20reported.20bug/near/320792613)
that had previously been discovered and fixed. This is exciting, and
within the Cranelift project we plan to do whatever is needed to
facilitate a full verification of our backends (starting with their
aarch64 focus) and integrate their work into our tree.

The initial verification project has been scoped to ensure
feasibility: backend rules only (not mid-end), for aarch64 only, CLIF
opcodes used by Wasm lowering only, and integer only. In due time, we
hope that we can relax each of these. Our collaborators have looked
into some x86-64 semantics that might be used to verify our x64
backend as well. Verifying floating-point lowerings is in some cases
(when it's not a trivial `fadd`-lowers-to-`fadd` rule) an open
research problem, but this would be great to have. And, lastly, one of
the reasons for our use of ISLE in the egraphs mid-end optimization
framework was to enable eventual verification of CLIF-to-CLIF rewrite
rules as well.

## Tail Calls

We have [begun planning support for tail
calls](https://github.com/bytecodealliance/rfcs/pull/29), and plan to
implement this fully in the first part of 2023. A "tail call" is a
call from one function to another in "tail position": that is, the
last thing that a function does before returning, such that the return
value of the callee becomes the return value of the caller. In this
case, there is nothing left to do in the caller, so we can actually
remove the stackframe entirely before *jumping* to the callee. Once we
have tail-call support, we have a generic way of implementing
arbitrary control flow between functions. As a result, tail calls are
the foundational form of control flow in functional languages such as
[Scheme](https://en.wikipedia.org/wiki/Scheme_(programming_language)),
and can be used as a very convenient building block by a compiler when
*targeting* a language as well. There is a [Wasm proposal for tail
calls](https://github.com/webassembly/tail-call) that recently
advanced to "stage 4" (the stage just before full acceptance into the
standard) so there is pressure from Wasmtime to implement tail-call
support in Cranelift. Once we have it, it will undoubtedly see use in
many different ways.

## Debugging

We have long had *partial* support for debugging in Cranelift: in
particular, the compiler understands "debug value labels" applied to
CLIF values, and provides metadata in return with the compiled machine
code to indicate where in the machine state these values reside. The
embedder can use this to implement higher-level debug observability:
for example, Wasmtime translates Wasm-DWARF debug info to native DWARF
so that a debugger attached to the Wasmtime process can step through
and view local variables and memory state at the Wasm level.

While this is nice to have, it has some longstanding issues, and no
one has been able to dedicate the time to give this part of the
codebase the attention it deserves. For example, [this failing
assert](https://github.com/bytecodealliance/wasmtime/issues/4669) is
reported semi-regularly; while the actual failure is in Wasmtime DWARF
code, it's unclear where the root cause lies. At the Cranelift level,
we sometimes [lose observability of
locals](https://github.com/bytecodealliance/wasmtime/issues/3884) and
[globals](https://github.com/bytecodealliance/wasmtime/issues/3439)
more often than we could with more work.

We have a general idea of what is needed to make the native-DWARF
debugging experience more robust (e.g., [this
refactor](https://github.com/bytecodealliance/wasmtime/issues/5537) on
the Wasmtime side, and whatever fixes it may need on the Cranelift
side). Beyond that, we have a number of ideas for debugging support at
different semantic levels, and hope to form a [Debugging
SIG](https://github.com/bytecodealliance/governance/pull/26) to
develop these ideas further.

## Exception-Handling

We have wanted to implement [exception-handling
support](https://github.com/bytecodealliance/wasmtime/issues/2049) for
some time. At the Cranelift level, this requires changes to how the
compiler reasons about control flow (to account for exceptional edges)
and mechanisms to produce the metadata needed by the unwinder as it
searches for handlers (catch-points). Such support would be
immediately useful to multiple embedding use-cases: the [Wasm
exception-handling
proposal](https://github.com/WebAssembly/exception-handling) will
require it, and so will proper unwind support in cg\_clif. Any other
language implementation built on Cranelift that requires zero-cost
exception unwinding could make use of this feature as well.

## Super-optimization (Achieving the Peepmatic Dream)

Now that we've
[implemented](https://github.com/bytecodealliance/wasmtime/pull/5382)
and [turned on by
default](https://github.com/bytecodealliance/wasmtime/pull/5587) the
egraph-based mid-end optimization framework, which allows us to
optimize expressions in the CLIF (program IR) by writing simple
rewrite rules in ISLE, we have several significant plans for using it
more fully.

One large project we've been hoping to return to for a while is the
use of
[superoptimization](https://en.wikipedia.org/wiki/Superoptimization)
to generate a large body of fruitful optimization/rewrite rules ahead
of time. The general idea here is that a compiler embodies domain
knowledge about the "best" way to achieve certain computations; we
should be able to precompute this domain knowledge by starting with
definitions of our building blocks (instructions or operators) and
doing a brute-force search. It doesn't matter too much how much time
this takes, because we only do it once, offline, to generate rules,
and then we check them in as part of the Cranelift source code.

We had previously worked toward this idea with
[Peepmatic](https://github.com/bytecodealliance/wasmtime/tree/dba74024aa412f284871375db292c1bf9079d769/cranelift/peepmatic),
and so in a very real way, the idea here is to "pursue the Peepmatic
dream". We believe we can reuse much of the infrastructure (e.g.,
tools to feed Cranelift program fragments into
[Souper](https://github.com/google/souper), a superoptimizer) and are
also interested to see what scaling challenges a large body of rules
may bring to the mid-end framework.

## Compiler Stats Framework

We would like to build a "statistics framework" for the compiler that
aggregates data about the compilation process in order to help with
debugging and performance tuning, and in general to improve compiler
observability. The basic idea is to instrument the various passes and
analyses of the compiler with updates to stats-counters for particular
"events" or cases -- for example, every time a redundant instruction is
removed by GVN, or a rewrite rule fires, or a register is spilled --
as well as measures of various data-structure and code sizes (number
of basic-blocks, instructions, register liveranges, etc).

Such data is very useful if it helps to narrow the potential causes of
an issue or of surprising performance. For example, when investigating
the compile-time impact of the register allocator, a very high counter
for "liverange splits" might strongly hint at what is going
wrong. Especially if there is a correlation between some statistic and
runtime or compile time, we might be able to quickly zero in on
performance bugs or places we could improve.

Objective measures like event-counts (e.g., number of register spills)
are also *extremely* useful for showing the effect of compiler
changes. They are in some sense the opposite of an "end-to-end"
measure like compile time or runtime. That is, these objective stats
are *deterministic* (identical every time for a given input),
*transparent* (if spill-count increases by one, we can actually
examine the one new spill) and *specific* (if spills increase, we know
the issue is with register allocation, or something that affects it
like register pressure). In contrast, end-to-end measures are *subject
to noise*, *opaque* and *often not actionable* (if measurements show
we got 0.1% slower, why and how do we fix it?).

We already have ad-hoc forms of this in the [egraph
framework](https://github.com/bytecodealliance/wasmtime/blob/20a216923b5d6bc129935ad56de1ca9ccea38949/cranelift/codegen/src/egraph.rs#L593-L612)
and
[regalloc2](https://github.com/bytecodealliance/regalloc2/blob/376294e828bd252aed717b6864fdcf9d637f23c8/src/ion/data_structures.rs#L658-L693). They
are not passed to the user in any systematic way, but rather just
printed to the log output as trace-level messages. We could start by
building a central framework that collects these numbers and then
provides them programmatically per compiled function. We could then
build *tooling* that allows printing stats for every compiled function
in a module (Wasm or CLIF), selecting functions with particularly
abnormal stats (e.g., "find me the function with the most register
spills / largest stack frame"), and *diffing* stats between runs.

## Support the Development of Winch (Baseline Compiler)

The [Winch](https://github.com/bytecodealliance/rfcs/pull/28) baseline
compiler project aims to provide an alternate backend (replacing
Cranelift) for the Wasmtime engine. While not within the scope of
Cranelift (hence this RFC) directly, the project *is* reusing pieces
of Cranelift. In particular, the back end of Cranelift -- the
`MachInst` struct for each ISA, its `emit` method, and the
`MachBuffer` -- form a reasonably usable assembler library, and Winch
has chosen to build on top of these abstractions to share effort with
us. This is mutually beneficial: not only does the baseline compiler
project get a large head-start, but the work has helped to clarify a
layering separation in Cranelift and will eventually drive more
refactoring to cleanly split this layer off. In the fullness of time,
this may be generally usable as a machine-code emission library. In
this RFC, we propose simply to dedicate the time to abstracting out,
exporting, or otherwise providing the pieces that Winch needs to
succeed.

## Miscellaneous Refactors and Improvements

### Legalizations in Mid-end Framework

A longstanding, slightly annoying aspect of our reworked backend
framework is that while *most* of the instruction selection and
lowering happens in a single pass while we generate VCode, there are
still some in-place mutations that happen beforehand. The
[legalizer](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/codegen/src/legalizer/mod.rs)
performs this rewrite with handwritten pattern-matching code --
perhaps the last place this kind of code still exists in Cranelift. We
should migrate these to the mid-end framework. This should not be too
complex, though it does require some thought about what to do in
non-optimizing mode (when the egraph pass doesn't run): perhaps there
can be a lightweight egraph-pass mode for "necessary"
rewrites. Building this would then give us another tool to use to
factor out common logic from backends: instead, "narrowing" (e.g.,
from i128 to i64 values) can be done by one machine-independent set of
rules.

### Other Fuzzing and Translation Validation?

In addition to the big-hammer "formal verification" project described
above, we should continue to plan and think of ways to fuzz and
validate correctness. For example, we have some ideas on a kind of
[translation validation for ISLE rule
compilation](https://github.com/bytecodealliance/wasmtime/pull/5435#pullrequestreview-1228022447),
to ensure that `islec` generates Rust code that faithfully implements
our validated rules (though in that specific case the idea may need a
little more thought to get to a practical effort/feasibility/benefit
tradeoff point). When we eventually [update and improve debugging
support](https://github.com/bytecodealliance/wasmtime/issues/5537)
(see below), one way of validating our handling of DWARF debug info is
to differentially fuzz against another debugger step-by-step. And so
on: there are likely many ways that we could build custom
oracles/checkers and validate compiler-specific properties, making
good use of the fuzzing time that we have.

## Documentation

Finally, we should make a concerted effort in 2023 to improve our
project documentation, in at least three different ways.

### Cranelift-Producer Examples (Toy Language)

We currently have fairly limited information available to new
Cranelift embedders. Ideally, the codegen library should be easy to
pick up and use, with examples to follow showing how to get started,
generate common kinds of control flow, memory access, calls to and
from host code, and the like. It may be easiest to show all of this in
the context of a toy language example implementation. Regardless of
the format, we owe our potential users better than the raw Rust API
documentation and barebones examples we have today. This carries
benefits back to the project, too, indirectly: making Cranelift easier
to pick up and adopt will lead to a greater number of users, which
provides us a more diverse and useful stream of feedback and a larger
pool of engineers who have a vested interest in seeing the compiler
improve.

### Cranelift Porting (How to Write a Backend)

We have had various inquiries over the past several years regarding
how to port Cranelift to target a new architecture. Our canonical
answer so far has been to refer to the other backends. This is only
barely adequate! We did receive a [very welcome contribution of our
RISC-V
backend](https://github.com/bytecodealliance/wasmtime/pull/4271), and
it is to the credit of the developer of that PR that it was built
without good documentation on our part. However, it would be far
better if we could provide a "How to Write a Backend" document. Note,
also, that while the *direct* audience of such a document is fairly
small, its indirect effect as a forcing-function for proper
documentation of backend architecture will also be quite helpful.

### Documentation Update Pass

Finally, we have a wide variety of documentation written over the
lifetime of the Cranelift project, and not all of it is up-to-date
with respect to the latest compiler design. We have been fairly
accepting of large compiler changes: a new backend framework, changes
to IR types (bools, flags, endianness) and instructions (branches),
the way optimizations are written, etc. While our current state is an
excellent starting-point, we should make a *full pass* over all
documents (standalone files and doc-comments) to ensure they are
up-to-date, accurate, and complete.

## Acknowledgments

This RFC contains ideas from discussions with Jamey Sharp, Trevor
Elliott, Nick Fitzgerald, Andrew Brown, and others. Thanks to all of
the contributors for the regular very stimulating and thoughtful
project meetings and the resulting inspiration as well.
