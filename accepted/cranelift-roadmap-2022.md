# Cranelift Roadmap for 2022

Around this time last year, we [published a roadmap for Cranelift in
2021](https://github.com/bytecodealliance/rfcs/pull/8). It has been a
productive year, and Cranelift is in a very different spot today than
it was at the end of 2020. (A separate "progress report" on 2021 is
forthcoming!) With all our work in the past year, we are now starting
to have what might be called a "solid foundation": we have worked to
improve many core parts of the compiler (e.g., ISLE, regalloc2), we
have paid down technical debt by completing a backend transition and
removing old code, we have significantly improved our backends
(especially SIMD) and our fuzzing/testing situation (differential
testing, the interpreter, custom mutator), and more. We will certainly
continue to improve these fundamentals, but one might also reasonably
ask: what are the next big steps we hope to take?

This document continues the yearly-roadmap tradition by documenting
our aspirations for 2022: general project directions, and specific
milestones, that we hope to reach.

## High-level Priorities

In a single sentence, the priorities and direction for Cranelift in
2022 can be summarized as: *improve the optimizer, make verification
efforts concrete, expand the feature set as needed, and let ongoing
work stabilize and mature*.

In a litle more detail, we identify the following five areas of work,
which we will expand below:

1. Complete the *ISLE*
   ([RFC](https://github.com/bytecodealliance/rfcs/pull/15),
   [compiler](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/isle/))
   and *[regalloc2](https://github.com/bytecodealliance/regalloc2)*
   efforts. These two projects improve the core compiler backend's
   generated-code quality, maintainability, and performance, and
   provide a solid foundation moving forward.
   
2. *Improve the compiler's [middle
   end](https://en.wikipedia.org/wiki/Compiler#Middle_end)*, or
   collection of IR-level optimizations. Cranelift has the basic
   optimizations such as constant propagation/folding,
   common-subexpression elimination, and dead-code elimination, but
   some of these are fairly basic, and other common optimizations
   (e.g., inlining) are missing; this will become more important in
   some expected future use-cases. As we do this, we should look for
   ways to facilitate verification efforts as well.
   
3. Add compiler support for *tail calls*, *exception handling*, and
   other features rquired by the Wasm frontend for upcoming proposals.
   
4. Continue to innovate in testing methodologies: add some randomized
   testing by instrumenting the backend to perturb compilation
   decisions ("chaos mode"), add additional fuzzing oracles and
   continue to improve fuzzing effectiveness with custom mutators,
   etc., and make use of ISLE's flexibility in rule ordering and
   rewrite application to improve testing coverage.
   
5. Achieve some concrete formal verification results. We have set up
   many of the necessary conditions for formal verification of parts
   of Cranelift's compilation/lowering pipeline by moving to a
   declarative description of lowering rules. We should start to find
   ways to apply, e.g., SMT-solver-based verification against
   externally provided IR and machine semantics, to lowering patterns;
   or to verify optimizations against metadata (side-effects,
   aliasing, etc.) of instructions; or similar techniques.
   
6. Clean up loose ends and unresolved design questions, and continue
   to polish off rough edges as the compiler matures.
   
7. Start to think about what a 1.0 release would mean: look for
   remaining holes, and plan for stability.
   
## Summary of Efforts and Confidence Levels

As in [last year's
roadmap](https://github.com/bytecodealliance/rfcs/pull/8), we provide
a summary table of project areas, individual tasks/milestones within
those areas. Each task/milestone is categorized as either *Concrete*
or *Ambitious*, depending on whether we know exactly what must be done
("just burn down a TODO list of mundane issueS") or not ("solve a
research problem"). In addition, each is classified with a confidence
level in completing the task in the coming year: *High* or *Medium*.

```plain
|-------------------------------------------------|------------------|
| Compiler backend                                |                  |
| - ISLE (migrate all lowering rules)             | Concrete, High   |
| - regalloc2 (use native SSA API)                | Concrete, High   |
|                                                 |                  |
| Improve optimizer/middle-end                    |                  |
| - Inliner                                       | Concrete, High   |
| - Improve classical optimizations               | Concrete, Medium |
|   - GVN, cprop, LICM, alias analysis            |                  |
|   - share bounds checks                         |                  |
|                                                 |                  |
| Features                                        |                  |
| - Security-mitigation features (CFI, ptr auth)  | Concrete, Medium |
| - Proper tail calls                             | Ambitious        |
| - Exception handling                            | Ambitious        |
|                                                 |                  |
| Testing and verification                        |                  |
| - Leverage ISLE to do randomized tests          | Concrete, Medium |
| - "Chaos mode" tester                           | Concrete, Medium |
| - Achieve some verification of ISLE rules       | Ambitious        |
|                                                 |                  |
| Project maturity                                |                  |
| - Clean up loose ends / unresolved design Q's   | Concrete, High   |
| - Revamp documentation                          | Concrete, High   |
| - Ensure compiler-core completeness             | Concrete, Medium |
|   (IR + lowering coverage, APIs, good perf)     |                  |
| - Define 1.0 criteria and work toward achieving | Ambitious        |
|-------------------------------------------------|------------------|
```
   
## Compiler Backend Performance and Quality: ISLE and regalloc2

The first, and most concrete, goal for Cranelift in 2022 is to
continue the work we have been doing to improve the compile time
(compiler performance), quality of generated code, and maintainability
of the codegen backend.

This work has been organized into two major efforts: ISLE, an
instruction-selector DSL; and regalloc2, a new register allocator.

Most of the implementation work for both of these projects occurred in
2021: regalloc2 is essentially complete as a standalone project, with
just its Cranelift ingegration pending; and ISLE was [recently
merged](https://github.com/bytecodealliance/wasmtime/pull/3506) with
ongoing work to migrate all of our instruction lowerings to the DSL.

### ISLE

We need to complete the migration of instruction lowering patterns
from our handwritten code into the ISLE DSL. This work has proved to
be relatively fast so far, and can be done in small pieces. This is
because, as described in the [pre-RFC
#13](https://github.com/cfallin/rfcs/blob/cranelift-isel-pre-rfc/accepted/cranelift-isel-pre-rfc.md),
the DSL was designed explicitly to allow gradual migration via its
close integration with the Rust code, both "vertically" (within a
single instruciton lowering, we can call Rust helpers) and
"horizontally" (separate instructions can be migrated
separately). Thus, the main objective while performing the migration
should be to retain parity: instruction lowering tests continue to
pass, or else warn us when small differences in emitted code occur
(and we can choose to accept them). Initial [performance tests during
fitzgen's integration
work](https://github.com/bytecodealliance/wasmtime/pull/3506#issuecomment-966504233)
showed no effect to slight speedups in compile time; we expect that to
remain true during the remainder of the transition.
   
As part of this work, we likely will want to think about replacing a
bit more of the repetitive backend code with generated code. Right
now, for example, each instruction type (the type that implements
`MachInst` for each architecture) is a large Rust enum and some trait
methods for fetching and rewriting register metadata, pretty-printing
a disassembly, and the like. If we generated this information from a
table of metadata (e.g. a checked-in TOML or YAML file), the result
would be easier to use and less error-prone. Then the only handwritten
code should be (i) the emission code for each instruction format (we
do *not* want to revert to Rust-code-in-strings that solved this
problem in the old backends' DSL), and (ii) handwritten helpers for
things like immediate formats and type predicates, invoked from ISLE
as extractors.
   
Once we have fully migrated the lowering patterns, we have much more
freedom (i) to improve the lowerings, and (ii) to change the
underlying infrastructure; that leads us to our second point.

### regalloc2

We need to make use of regalloc2 in order to unlock compile-time
(compiler performance) and code-quality improvements.
   
regalloc2 was initially designed with a native API that accepts
SSA-based code. This is a fairly impactful design choice in many
details of the allocator. It means that the embedder (that's
Cranelift) can express register constraints, and phi-nodes (or
blockparams) from its SSA IR, directly to the allocator; in contrast,
with the regalloc.rs API, the lowering code in Cranelift translates
blockparams into sequences of move instructions, and also translates
register constraints (e.g. at callsites or certain instructions) into
move instructions, leading to lengthier code that needs to be cleaned
up later (by move elision). Thus, we expect that the SSA-based design,
with unified handling of moves between CLIF's blockparam semantics and
regalloc-induced loads/spills/moves, should lead to better
performance.
   
regalloc2 also includes support for live-range splitting: unlike
regalloc.rs, the new allocator can make different allocation decisions
during different phases of a function's execution, so that, e.g., a
value that is very hot during one inner loop and live (but inactive)
over a prior inner loop can be spilled temporarily, then brought back
to a register for fast access. In contrast, regalloc.rs, once it
decides to spill a value, must generate loads/stores to the spillslot
each time the value is read or written.
   
Unfortunately the [aborted attempt at porting Cranelift directly to
this SSA API](https://github.com/cfallin/wasmtime/tree/regalloc2) was
more work than expected: it was left on May 4 with a commit "down to
547 type errors!" (eek!) after ~4KLoC of diffs. So we built a
[compatibility
shim](https://github.com/bytecodealliance/regalloc.rs/pull/127) to
emulate the regalloc.rs API, after adding support for non-SSA code and
a lot of move-related optimizations to regalloc2. Unfortunately,
finding someone with the time and expertise to review that has been a
challenge. (It's probably a week or so of work, but requires deep
understanding.)
   
So the current plan is to wait for ISLE to land, then use the ability
to modify the generated backend code to *systematically* transition to
SSA. To do this, we can leverage parts of the above patch to the
MachInst and ABI framework code, and then change a few details in the
ISLE glue. We can be much more confident in maintaining correct
codegen than we would if we had rewritten all of the lowering patterns
to the new register types and SSA constraints by hand.

### Improve Instruction Lowerings Using ISLE
   
Continue to iterate on the instruction lowerings and improve quality
(and where applicable, correctness). Now that we have a framework that
makes this easy, we should take full advantage of it! The tireless
backend hackers amongst us have given significant useful feedback on
the DSL and it will be exciting to see how the backend quality can
improve even more.
   
A particularly valuable concrete milestone would be to stabilize our
SIMD support and turn it on by default. Beyond that, the sky is the
limit: any sort of special-case optimization that would have been more
difficult to express in the handwritten pattern-matching code can now
be added in a few lines in the DSL.
   
## Improved Optimizer (Compiler Middle-End)

Second, Cranelift needs to improve its "middle end", or
machine-independent optimization stage. Most of our effort in the past
several years has been in improving the backends, but our "standard
suite of optimizations" -- GVN, DCE, LICM, constant
propagation/folding, and the like -- have not changed in that
time. This has served us perfectly fine for a while, but both new
workload characteristics, and increasing focus on the parts of the
pipeline that we haven't recently optimized (given all the work on the
backends), indicate that perhaps this should change.

In particular, the WebAssembly MVP design, which drove Cranelift's
design, implies some interesting code characteristics. Because a Wasm
module is typically (not always!) produced by an optimizing compiler
such as LLVM, heavyweight optimizations are *usually* not very
worthwhile. For example, trivial helper functions will have already
been inlined before the Wasm module was produced, so inlining is less
important for a Wasm engine to implement. Similar arguments can be
made for heavyweight strength-reduction and algebraic-identity
optimizations, constant propagation, LICM, and the like. Translation
from Wasm to CLIF *does* introduce some redundancies, for example in
heap address computations, but Cranelift's simple optimizations can
handle many of these issues.

However, multi-module WebAssembly workloads will become more common as
[Module Linking](https://github.com/WebAssembly/module-linking) is
used by real workloads and as tye [Interface
Types](https://github.com/WebAssembly/interface-types) idea becomes
real. This changes everything: when we have a late-binding
environment, in which two Wasm modules that have never met each other
before are linked together due to user configuration, all of the usual
compiler optimizations may apply to this new combination of code.

### Inlining

An inliner is a necessary prerequisite to any optimizations across
modules; otherwise, we can't take advantage of any knowledge that
derives from the combination of modules (unless we do interprocedural
analysis, which we are unlikely to ever do).
   
So, we should develop a simple inliner, with the unique design
constraint that it only needs to work well in cross-module
contexts. We will likely need a reasonable heuristic to do the usual
"inline the helper functions, but not too much" thing, but we also can
likely be a little more conservative, in order to avoid pessimizing
already-carefully-optimized individual modules. In particular, we use
Wasm-specific information if we have it to only consider Interface
Types-related callsites, or cross-module callsites, or something to
that effect.

### Revisit Classical Optimizations
   
With the above done, we can revisit LICM (loop-invariant code motion),
GVN (global value numbering, a kind of common subexpression
elimination), DCE (dead-code elimination), constant
propagation/folding, and others. We have functional implementations of
each, but we suspect that some of the details could be improved.

### CLIF Interpreter for Constant Optimizations
   
In particular, now that we have the [CLIF
interpreter](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/interpreter/)
implemented by Andrew Brown and expanded substantially by afonso360,
we could potentially use this to perform constant folding, and maybe
other optimizations. Andrew has done some exploratory work in this
direction and it could be interesting to see what benefits it can
bring.

### CLIF-to-CLIF Peephole Pass

As another use-case for the ISLE DSL, with slightly different glue: we
could enable CLIF-to-CLIF transforms as well as the current
CLIF-to-MachInst ones, and use this both for legalization and for
architecture-independent optimizations ("peephole" patterns, algebraic
identities, etc.). This [continues the Peepmatic
mission](https://github.com/bytecodealliance/wasmtime/pull/3543#issue-1056639878),
as fitzgen expressed in that PR, and it's exciting to imagine where
this could lead.
   
This will likely be best-implemented as a seprate pass at first,
because we want the second (CLIF-to-MachInst) pass to be able to match
across the individual rewrite results of the first pass, implying some
intermediate buffering; however, composing the two passes inline in
some way might also be interesting, if we can work out a
DSL-compiler-level way of doing this composition efficiently.

## Support for Tail Calls, Exceptions, and Other Features

There are several compiler features that we do not yet support but
will eventually be needed by at least the WebAssembly front end, and
likely other consumers as well. These are features that have
implications throughout the middle- and back-ends, and cannot be
"polyfilled" (or at least not easily in a low-overhead way) on top of
today's CLIF semantics.

### Proper Tail Calls

Proper tail call support is a common programming-language feature: for
example,
[Scheme](https://en.wikipedia.org/wiki/Scheme_(programming_language))
mandates that an implementation support this feature. Among
Cranelift's common uses-cases, WebAssembly has a [proper tail
calls](https://github.com/WebAssembly/tail-call) proposal that is
currently in a late stage, waiting for more implementations to come
online.

A tail call occurs when a call instruction is the last instruction in
a function body, and the result of the call is directly returned. That
call is then known as a "tail call" and can actually be replaced with
a jump. This way, the tail-called function can return directly to our
caller, saving the side-trip back through the current function. Once a
language or compiler IR has tail calls, they can actually be used in
many ways: for example, they are they primary control flow mechanism
in Scheme and are used to implement looping. When present in Wasm,
they become a very useful tool for compilers targetting Wasm bytecode.

The difficulty with implementing tail-call support comes in the
*uncommon cases* (as it does for so many compiler features). In other
words: when everything goes right, a tail call is just a jump, after
putting the arguments in the correct registers. But if the tail-called
function has stack arguments, and if we pass some of them by loading
values from our stack spillslots, and if we have to remove our
stackframe before making the call (a requirement of proper tail-call
support to ensure that an infinite sequence of tail-calls does not
grow an unbounded stack nest), then we quickly have a general "state
permutation" problem with difficult corner cases. [This GitHub
comment](https://github.com/bytecodealliance/wasmtime/issues/1065#issuecomment-914420622)
describes the problem in a little more detail.

To implement this, we'll need to implement a new internal ABI in which
callees clean up their stack frames; and then we'll need to implement
the tail-call operator itself, including the corner-cases described
above. The primary Wasm-engine consumer of Cranelift, Wasmtime,
already uses trampolines to bridge between Wasm and host-ABI code, so
a new ABI is not a large problem; getting the corner cases right is
the hard part!

### Exception Handling

Another language feature that requires explicit compiler support is
*exception handling*: the ability to unwind stack frames and perform
cleanup along the way, stopping at relevant handlers, all without
overhead in the common (non-exception-throwing) case.

Zero-overhead exception support requires several things from the
compiler. First, it must emit metadata that correlates locations in
the program (PC values) to relevant handlers. It must also emit
precise metadata that enables unwinding of stack frames from any point
that might experience a thrown exception. Finally, exceptional control
flow must be taken into account by optimizations.

We will likely refer to existing designs here, for example [LLVM's
exception
handling](https://llvm.org/docs/ExceptionHandling.html). That said,
our initial design should be driven by what is necessary to support
the [Wasm exception handling
proposal](https://github.com/WebAssembly/exception-handling), keeping
in mind more general use-cases to ensure we make the design
future-proof.

We have [started to
discuss](https://github.com/bytecodealliance/wasmtime/issues/3427)
this topic recently; initially we had been converging on possibly a
simple purpose-built unwinder, but after more discussion in [a
recently Wasmtime biweekly
meeting](https://github.com/bytecodealliance/wasmtime/blob/main/meetings/wasmtime/2021/wasmtime-10-28.md),
it seems likely we will stick to platform-based support using DWARF
(and eventually Windows Structured Exception Handling) for now.
   
### ISA Security Features: Control-Flow Integrity and Pointer Authentication

Instruction-set architectures in recent years have added features that
are meant to improve application security. These hardware-level
security mitigations include means to enforce [control-flow
integrity](https://en.wikipedia.org/wiki/Control-flow_integrity) and
to guard against mangled pointers.

For example, AArch64 introduced Branch Target Indicators (BTI) in
ARMv8.5-A, and x86-64 introduced Control-flow Enhancement Technology
(CET). Both of these ISA features provide a means to ensure that
branches cannot be maliciously redirected. First, branches are
restricted to branch only to actual basic blocks: `ENDBR` ("end
branch") instructions on x86-64, or `BTI` instructions on aarch64,
allow a branch to that instruction to succeed (all other branches fail
in protected regions). Then, either a shadow stack (x86-64) or pointer
authentication (aarch64) prevent return addresses from being
clobbered.

AArch64 also includes functionality to sign pointers and validate
them. This feature, called Pointer Authentication, mangles the
otherwise-unused high bits of critical pointers (such as the stack
pointer) using some key, and un-mangles them before use. Any attempt
to corrupt critical state will likely result in dereferences of
mangled pointers instead, which safely trap.

We have the open [RFC
#17](https://github.com/bytecodealliance/rfcs/pull/17) that proposes
using PAuth and BTI in the aarch64 backend. We should move forward
with this and the equivalent implementation on x86-64, and enable its
use whenever the hardware and embedding environment support it.
   
## Randomized Testing

We have made significant use of various forms of randomized testing to
improve Cranelift's correctness. The most common sort of randomized
testing is fuzzing, and we actually have a rich history of
differential-execution fuzzing of Cranelift, with four (!) fuzzing
setups developed by four separate people. These are: fuzzing at the
Wasm level against wasmi in [PR
#2543](https://github.com/bytecodealliance/wasmtime/pull/2453), at the
Wasm level against the Wasm spec interpreter in [PR
#3124](https://github.com/bytecodealliance/wasmtime/pull/3124),
at the Wasm level against Chromium's V8 in [PR
#3264](https://github.com/bytecodealliance/wasmtime/pull/3264)),
and at the CLIF level against the CLIF interpreter (in [PR
#3038](https://github.com/bytecodealliance/wasmtime/pull/3038)).
No one can reasonably claim that we don't believe in differential
fuzzing! We also have fuzz targets that test, e.g., just compilation,
the wasm-mutate custom mutator, and other aspects of the compiler.

Given this history, then, what is left to be done? The largest
opportunity that remains is, likely, in more "application-specific"
forms of randomized testing. In other words, rather than drive a
compilation and some oracle with a random Wasm module, we should more
directly randomly perturb the compiler by instrumenting and
controlling other parts of it. This way, the "reach" of the randomness
is much greater (i.e. we should expect better test coverage for the
same effort).

### Pervasive "Chaos Mode" in the Compiler

The first major direction to pursue is to instrument the compiler's
logic to allow for a "chaos mode" in which, at various decision points
where the compiler has multiple valid choices, the choice is driven by
some "randomness context" that can be controlled externally.

To give some concrete examples: basic-block ordering, specific
register allocation choices, whether or not to do certain
optimizations, and which of several possible instruction lowerings to
pick, can all be randomly perturbed without affecting the correctness
of the compilation. A traditional fuzzing-based testing methodology
would attempt to find bugs in these parts of the compiler by searching
for an input program that triggers a particular combination of
circumstances in the relevant compiler stages. But if there is direct
freedom in many choices in the backend, why not try turning those
knobs?

The intuition is that, the closer we have such a free-to-turn knob to
the compiler state that can trigger a bug, the less of the compiler
the fuzzing engine has to reverse-engineer to find a suitable
bug-triggering input. Said another way: fuzzers are great at finding
inverse functions but it's still costly; better to remove the
indirection in order to multiply their power per unit of time spent.

In addition, not all of the state space in intermediate portions of
the pipeline may be reachable, or easily reachable, by choosing some
input program. Perhaps an earlier compiler pass has some normalization
effect that masks attempts at exploring the space. Or perhaps it is
just very costly (e.g.: a long sequence of operations that causes a
particular spill pattern) to reach some of the configurations we want
to test.

So, seen in that way, providing "chaos mode" instrumentation
throughout the pipeline should give us a unit-testing-like
factorization that allows bugs in each stage of the compiler to be
independently exposed.

The list of potential knobs to turn is long but a few that are likely
to be reasonably easy to implement are:

* Randomly spill or reload a value in the register allocator.
* Randomly clobber some state that should be undefined: high bits of a
  register, or an unused spill-slot or register, or caller-saves at
  the start of a prologue, or callee-saves at callsites.
* Randomly reorder code.
* Randomly skip merging loads with ALU operations, but do the load
  again at the ALU operation and compare.
* Randomly choose different instruction lowerings (see below!).
* Randomly disable certain optimizations, independently at each
  potential optimization site. Applicable in the middle-end for each
  optimization pass (e.g.: randomly do not hoist a particular loop
  statement in LICM), and in backend infrastructure, e.g. randomly do
  not let instruction selection see further up an operand subtree.
  
Some of these transforms (checking ALU op + load consistency,
clobbering state that should be unusable) can be enabled
unconditionally as well; they then fit into a sort of "address
sanitizer" or "debug asserts" sort of philosophy, where extra checks
are always performed.
     
### Randomized ISLE Rule Application

Specific to the randomization of the backends, we can leverage our
ISLE DSL in new and interesting ways to *systematically* test the
self-consistency of each backend's rules.

Term-rewrite systems such as ISLE have a fundamental "many-paths"
property: whenever more than one rewrite rule could apply, then there
are many possible ways to resolve the instruction selection problem,
landing at many possible final answers. Determinism is provided by
some rule-ordering heuristic or user-provided priority, but rewrite
rules need to be correct at least independent of the heuristic. We can
leverage this possible nondeterminism (or DSL compiler-specified
determinism, rather) for testing.

The first idea, then, is this: either modify the ISLE compiler to take
a random seed at Rust-code-generation time and reorder its
decision-trie edges appropriately, then test the resulting compiler
backend for execution correctness, or else (harder but likely
possible) take some runtime randomness to choose between multiple
differently-ordered tries (internal constructor bodies for the
lowering constructor terms) in order to randomize on a per-lowering
basis.

The second idea is to *omit* rather than simply randomize the
application of rules. We could either mark some lowering rules as
"optional", indicating that the compiler should still be "complete"
without them, or else just randomly delete rules and then discard
fuzzing runs where the compilation fails. (ISLE's support for
partial-functions at the top level should allow a clean exit in order
to implement this.) Then, for N rules we could exclude, we hvae 2^N
compilers to test.

These ideas both essentially are ways to produce many compilers from
the single DSL description of the lowering, such that if the DSL
description is correct, then all compilers should produce code that
yields the same *execution results*. If this does not happen, then
there is a bug either in a lowering rule or in the DSL
compiler. Testing this property would likely give us a fairly strong
assurance in the self-consistency and robustness of the lowering
rules, which is orthogonal to the "correctness against a Wasm/CLIF
semantics spec" property tested by our usual differential fuzzing, but
just as valuable.
     
### Fuzzing with Custom Mutators

There has been substantial and very innovative progress recently in
developing
[wasm-mutate](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasm-mutate),
a custom mutator that, given a Wasm module and some random state,
produces a new semantically-equivalent module. This allows a fuzzing
engine such as libFuzzer to more easily span the space of test cases
by jumping out of local maxima, and is a very useful tool in ensuring
the completeness of our compiler testing. We should continue to
develop this and other Wasm-level mutators in the future to improve
our fuzzing effectiveness.

## Formal Verification

We have discussed our goal of [formal
verification](https://en.wikipedia.org/wiki/Formal_verification) of
parts of Cranelift for some time now. In the [RFC for
ISLE](https://github.com/bytecodealliance/rfcs/pull/15), for example,
we held verification of instruction lowering as a first-order goal to
be enabled by the DSL design. Some of our testers and analyses, such
as the [regalloc
checker](https://github.com/bytecodealliance/regalloc.rs/blob/main/lib/src/checker.rs)
and [VeriWasm](https://github.com/PLSysSec/veriwasm), can be seen as a
"translation verification" system, which is a weaker (per compilation
rather than once-and-for-all) kind of verification.

In the next year we should attempt to find some concrete ways to
achieve formal verification of some part of the compiler. The
instruction lowerings are the obvious choice, now that we have ISLE.

The most plausible path is to start by isolating each instruction
lowering pattern, tying its left-hand side and right-hand side to
known semantics, and showing that certain properties are preserved
through the transformation. For example, side-effects such as memory
accesses or possible traps, and the bit-level semantics of any
result-value computation, can be translated to some formal
representation (such as SMT-solver clauses) on both sides of the
lowering and then equivalence can be proved.

There are possibly interesting ways for middle-end optimizations to be
validated against a semantic description of the IR, too: for example,
code-movement could be validated against side-effect information, and
peephole/folding rules could be validated formally as well.

If we build a tool of this sort, then we should work to integrate it
into the normal development workflow as early as possible (as long as
it does not impose undue burden). For example, it would be wonderful
if a CI job could verify, for each PR, that newly added instruction
lowering rules did not introduce unexpected side-effects or corner
cases with incorrect results.

## Cleanups, Loose Ends, and Rough Edges

There are a number of "in-flux" portions of Cranelift's design and
implementation that need to be cleaned up, or at least audited, now
that the high-order bits (backend migration, instruction selection
framework, ...) are starting to solidify.

Now that we have developed and landed many of the major pieces that we
have wanted to build -- new backends, a new instruction selector
framework, support for major Wasm features like SIMD, etc. -- we have
a little bit of breathing room to polish what we have and resolve the
smaller issues.

### IR Semantics

Some of the loose ends and rough edges are semantic-definition or
design questions that were never properly nailed down. Cranelift is
still a young project in some ways, and the pieces that serve as the
foundation for its more common use-cases -- for example, the parts of
the IR and compiler used by the Wasm frontend -- are more fully
defined and have fewer incomplete portions than other corners do.

We can always continue to decide questions and nail down semantics "on
demand", as other use-cases arise, and this is a very helpful thing:
for example,
[rustc\_codegen\_cranelift](https://github.com/bjorn3/rustc_codegen_cranelift)
stresses Cranelift in different ways than `cranelift-wasm` and
Wasmtime do, by emitting different CLIF instructions and embedding the
compiler in a different way, and this has led to a number of valuable
PRs to expand the supported domain of CLIF code.

But we should also think about this in a more systematic way: before
Cranelift can claim anything like a 1.0 status, we should ensure that
the intermediate representation, CLIF has a clean and orthogonal set
of operators and datatypes, and that all reasonable combinations are
supported. Likewise, we should audit the APIs that generate and
manipulate it to ensure that the public interface is sufficiently
flexible and future-proof.

First, there are many corners of CLIF that are not used by the Wasm
frontend but are nevertheless present in the IR. These include:

- boolean types
- narrow-integer (i8, i16) and wide-integer (i128) types
- various kinds of flags-consuming and flags-producing arithmetic
  operations
- platform-specific primitives such as TLS (thread-local storage) and
  different kinds of relocations
  
At the most basic level, we need to ensure that all opcodes support
all relevant types. The CLIF-level fuzzer has done a reasonable job of
testing this, but we should also check that the allowed types in the
CLIF opcode definitions themselves are reasonable.

There are also semantics-design questions that may have significant
non-obvious ramifications in (i) specific machine backends (we should
strive for efficient mappings on all architectures), (ii) producers
and (iii) optimization passes. For example:

- The underlying bit-level storage of boolean types has an ambiguous
  or at least under-documented specification
  ([#3205](https://github.com/bytecodealliance/wasmtime/issues/3205)). There
  are non-obvious implications to the efficiency of SIMD code
  (masking, mainly), and subtle implications regarding normalization
  and undefined behavior when we discuss casting into (or loading from
  memory into) first-class bool types, that need to be more fully
  discussed.
  
- The `iflags`/`fflags` mechanism needs a rethink
  ([#3249](https://github.com/bytecodealliance/wasmtime/issues/3249))
  to ensure that our IR is working at the appropriate abstraction
  level. It's possible that another approach (e.g., generic `bool`
  flags, and appropriate pattern-matching for efficient codegen) is
  better; we need to discuss!
  
- There are sticky questions around what it means for CLIF to have a
  platform-independent meaning that have been raised by a combination
  of big-endian systems and the CLIF interpreter
  ([#3369](https://github.com/bytecodealliance/wasmtime/issues/3369)). This
  is a good example where an earlier compromise ("platform-specific
  endianness" option on loads and stores) has come back to bite us
  once it is dropped into a new context (a "platform" that is separate
  from any ISA). In such cases, we really do need to "get it right":
  the alternative, of semantics that are always plagued with some
  exceptions ("deterministic except for..."), leads to endless
  headaches and subtle correctness issues later.
  
- Similarly, there are some semantic mismatches where pieces "just
  happen to fit" now and we need to do something better: for example,
  the backends have a notion of
  [`RelocDistance`](https://github.com/bytecodealliance/wasmtime/blob/d1e9a7840eff310d77c253152129e7d963735382/cranelift/codegen/src/machinst/lower.rs#L307-L317)
  that is used to determine what kind of relocatable code or data
  reference to make, and we use fuzzy terms like "near" and "far" and
  carefully ensure that the actual use cases (e.g. Wasm modules) fit
  within actual implementation details (e.g. AArch64's +/- 128MiB jump
  offsets). A more proper solution is possibly to have a notion of a
  "memory model", as that term is used in ABI specifications, and to
  refer to that memory model in the compiler options and/or the IR.
   
### Documentation
   
It has been some time since we have updated the Cranelift
documentation in
[`cranelift/docs/`](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/docs/). Some
of the details here refer to the old backend framework (now removed),
while others are just obsolete, and moreover, there are many new
aspects to the design that are documented in rustdoc comments, RFCs,
or GitHub issues but not separate and more formal documentation. We
should make a concerted effort to bring our documentation up to date.
   
## Maturing Toward 1.0

Finally, a modest proposal: 2022 may be the year in which Cranelift
becomes well-baked enough to reach a 1.0 release milestone. This is
not at all a promise or commitment, but an aspiration, and an
invitation to consider more deeply: what do we need to do to make this
happen?

First, we need to build a list of criteria to judge when we are "done
enough" to make such a release. This should include at least:

- Functional completeness. This includes major consumer-level features
  that we think should be included (stabilized SIMD?), compiler
  features we think are necessary to be considered complete
  (optimization passes, tooling and metadata integrations, etc.), as
  well as comprehensive completeness in the details of each thing we
  implement (e.g. supporting all combinations of operand types for
  each instruction).
  
- High testing coverage and defense in depth. We want to have high
  confidence that we don't have severe codegen bugs. The best way to
  ensure this is to build out a diversity of testing strategies, as we
  have been doing: fuzzing in various ways, a large static test suite,
  actively supporting multiple users/embedders and fielding any issues
  they discover, etc.
  
- Documentation quality. We need to ensure that every major concept,
  API, design invariant, or "generally known thing" about Cranelift is
  documented so that users do not need to come to our Zulip and
  manually pick our brains (though they are always welcome to do so!).
  
- Compiler architecture and API stability. We should ensure that we
  are reasonably happy with our current interface (APIs) and overall
  design.
  
  We should audit all public API surface to ensure that we're happy
  with what is exported, that we are not locking ourselves into
  backwards-compatibility issues later (or a need for unnecessary
  semver breaks), and that there are no obvious holes or
  incompleteness.
  
  We should consider our overall design in this light, too. We can
  always continue to evolve, and we most likely will; but we should
  not reach a major release milestone with a pending consensus like
  "but we'll need to replace component X soon because it's really not
  right...".

- Process for releasing and security. Part of being a mature software
  project is having process that provides structure to workflows,
  removing some possibility of error and removing cognitive burden
  (especially in stressful times, such as emergency security
  releases). Right now we piggyback on the Wasmtime process for
  releases, by virtue of living in the same repository, and there is
  large overlap with their security processes (any codegen security
  issue will likely be a Wasmtime security issue and so will push
  through the necessary patches, including to Cranelift, under their
  process). Continuing to participate at this level is probably a
  reasonable option, but we should explicitly decide, and make a
  statement about expanding the guarantee, e.g. patches to all
  Cranelift security issues even if Wasmtime is unaffected.

- Design guidelines and principles. To guide further work, a mature
  software project should have a published set of design guidelines,
  principles, and philosophy ("why do I exist?"). We have done some
  thinking on this topic for Cranelift specifically -- what makes this
  compiler different than all the others, and what do we want our
  focus to be -- but we should publish a document to help guide future
  evolution and provide a framework for deciding which future
  directions are in-scope.
