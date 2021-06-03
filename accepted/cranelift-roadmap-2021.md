Cranelift Roadmap: 2021 and Beyond
==================================

Given the progress we've made on developing Cranelift in 2020 --
adding two new backends (AArch64 and a faster x86-64 one), a new
instruction-selection / lowering and code emission design,
incorporating a new co-developed register allocator (regalloc.rs),
adding codegen-level support for Wasm proposals such as reference
types, adding continuous fuzzing of our codegen and fixing bugs along
the way, and many smaller things -- there are many possibilities for
further work to be done in 2021. This document attempts to identify
our main priorities, and discusses some of the work items that we are
considering in order to enhance Cranelift in the next year.

High-level Priorities
---------------------

At the highest level, we have four priorities for Cranelift:

1. Complete the effort to migrate to the new backend framework, and
   deprecate and eventually remove the old x86 backend without
   regressions in functionality.

2. Work to ensure correct compilation output as much as possible.

3. Improve performance, to the point that Cranelift is competitive
   with state-of-the-art JIT engines (e.g., SpiderMonkey and V8) in
   compiler throughput and in performance of generated code.

4. Consider ways of improving security and providing options for
   hardening generated code (beyond just ensuring correctness).

Confidence Levels
-----------------

We describe a number of projects below that range from "relatively
straightforward" (backend migration) to "active area of research"
(compilation-result verification ideas). The amount of work required
for the tasks varies widely as well.

Given that, we will categorize the tasks into three categories based
on how confident we are that we will be able to achieve them:

- *Concrete project, high confidence of completion*: We know that the
   work is possible, with no major unknown design questions or
   unsolved technical problems. We are very likely to have the
   necessary time to complete it in 2021.

- *Concrete project, medium confidence of completion*: We know the
  work is possible, but we prioritize it slightly lower than the "high
  confidence of completion" projects; depending on resources and
  available time, we may not complete all of these, though we hope to
  do so.

- *Ambitious or aspirational goal*: We have a general sense that a
  project could be possible, and would provide significant benefit if
  completed; but we have not yet worked out all the details, or some
  technical problems require more thought or investigation. We list
  the idea here to map out where we would like to go, if we can.

Here is a summary of the below tasks with confidence levels:

```plain
    |---------------------------------------------------+------------------|
    | Backend migration                                 |                  |
    | - Remaining feature support in new backend        | Concrete, High   |
    | - Transition Lucet                                | Concrete, High   |
    | - Transition Wasmtime                             | Concrete, High   |
    | - Transition cg_clif                              | Concrete, High   |
    | - Remove old x86 backend                          | Concrete, Medium |
    |---------------------------------------------------+------------------|
    | Correctness                                       |                  |
    | - Expand use of fuzzing (beyond current)          | Concrete, Medium |
    | - Code generation for lowering                    | Ambitious        |
    | - Investigate verification ideas                  | Ambitious        |
    |---------------------------------------------------+------------------|
    | Performance                                       |                  |
    | - Establish continuous benchmarking               | Concrete, High   |
    | - Improve codegen quality                         | Concrete, High   |
    | - Code generation for lowering (perf exploration) | Ambitious        |
    | - VCode as a machine-independent IR               | Ambitious        |
    |---------------------------------------------------+------------------|
    | Security                                          |                  |
    | - Investigate use of ISA extensions (ptr auth))   | Concrete, Medium |
    | - Consider other compiler mitigations             | Concrete, Medium |
    |---------------------------------------------------+------------------|
```

Backend Migration
-----------------

The most concrete major goal for Cranelift in 2021 is to complete the
backend transition. Beginning in early 2020, we developed a new
framework for instruction selection and lowering and binary code
emission that is structured around `MachInst`s and `VCode` containers;
the latter is a linear, register-oriented (non-SSA) container of
lowered `MachInst`s, i.e., a machine-code oriented IR that is separate
from CLIF. We now have functional AArch64 and x86-64 backends in this
framework, but we continue to support the existing x86 backend as well
(and presently, it is the default backend on x86).

The transition will bring several benefits:

- The *performance* of generated code will increase. Measurements at
  various points in time have shown 10-15% performance increase,
  depending on the benchmark and the embedding in which Cranelift is
  used. Compilation is also generally faster, sometimes substantially
  so, due to algorithmic improvements in our new register allocator
  relative to the current one.
  
- The *maintainability* of Cranelift backends will increase. The new
  backends are generally written in a more direct style, with less
  indirection and "magical" code generation, than the old system that
  was based on recipes, encodings, and repeated application of
  legalization rewrites. Feedback from contributors working on new
  backends has been positive, and the relative ease of development has
  enabled support for at least two new architectures to be in various
  stages of development (ARM32, partial support in-tree; and IBM Z,
  not yet in-tree but running full Wasm MVP).
  
- *Build time* of Cranelift will reduce when we no longer have
  substantial amounts of generated code produced by the `meta`
  crate. This is important in projects like Firefox that have long
  build processes and large CI configurations that build Cranelift
  many times per day.
  
- We will be able to *remove substantial amounts of legacy code* once
  all compilation passes through the `MachInst` infrastructure, and
  the path will be cleared for additional compilation-performance
  ideas described below.
  
The transition will likely occur in several steps:

1. *Transition users* of the current x86 backend over to the new
   backend individually, and ensure that nothing breaks. These users
   include at least Wasmtime, Lucet, and rustc\_codegen\_clif
   (`cg_clif`), and possibly other stakeholders TBD. This can be done
   by enabling the `experimental_x64` feature when depending on the
   codegen crate.
   - Lucet is known to work already, and the new backend was recently
     added to CI tests (bytecodealliance/lucet#616).
   - Wasmtime will require a few features to be implemented (below).
   - `cg_clif` will require a few features as well (below).
   
2. *Switch the crate default* so that all users use the new backend;
   provide a feature to select the old backend if any other users have
   compatibility issues.
   
3. *Remove the old backend*.

4. Start to *remove dead code* left over, and simplify
   accordingly. Legalization can be greatly simplified (it becomes
   just an expansion of certain high-level opcodes; all backends
   accept the same core CLIF opcodes), and quite a bit of plumbing can
   be removed.

In order to transition Lucet, we can simply flip the feature flag in
the codegen crate dependency whenever we are confident to do so.

In order to transition Wasmtime and cg\_clif, we need:

1. Full *debug-info support*. We have some unwind info generation in
   the new backends, which is usually sufficient for libunwind to walk
   the stack; and we have source location-to-PC maps; but we do not
   yet provide a mapping from program values (e.g., Wasm expression
   stack slots or locals) to registers or spillslots via DWARF.
   
2. Complete support for *SIMD*, at least up to the Wasm SIMD proposal
   level. This is backend-specific. The AArch64 backend meets this
   goal, but the x64 backend is solving a few last issues (see
   e.g. bytecodealliance/wasmtime#2432).

3. Support for *128-bit operations*. This is an important bit of
   functionality in order to support at least `cg_clif` (and possibly
   other users). It relies on associating SSA values with multiple
   machine registers, and on machine backend support. A draft PR
   exists (bytecodealliance/wasmtime#2504); we need to clean this up
   and finish the last bits needed by `cg_clif`.
     
4. *Windows Fastcall* calling convention support. This one is likely
   not too hard; it just needs to be done, by someone with a Windows
   environment. Note that the Wasm VMs that use Cranelift (Wasmtime,
   Lucet, Firefox's SpiderMonkey) generally use their own ABI, even on
   Windows, but fastcall support is important for use-cases that
   generate code that is externally exposed with a standard ABI.
   
It is always possible that other issues will arise as we transition to
the new backend, and we will handle those as they arise; but the above
are (to our knowledge) the only missing *feature* support.

Correctness
-----------

Cranelift is used in contexts where generating correct code is
*essential*. In particular, when used to compile Wasm modules that
contain untrusted code, the sandboxing depends on correct execution of
the IR that implements bounds checks for heap accesses and function
calls (among other checks). Hence, we place high priority on
engineering Cranelift to minimize its bug rate.

We are already using *fuzzing* as a key technique in several ways. We
fuzz Cranelift's frontend by feeding it random Wasm modules. Recently,
we added a differential fuzzing target to compare execution of
Cranelift-compiled Wasm against another Wasm interpreter. We have been
fuzzing the new backend's register allocator using a symbolic
checker. All of these fuzz targets have found real, subtle bugs. (We
are also thankful for the fuzzing efforts of projects that embed
Cranelift: in particular, the Firefox security team has done a
fantastic job ensuring that Cranelift is well-covered by fuzzing and
secure in that context.)

Going forward, we want to continue to minimize bugs in three ways:

1. *Expand fuzzing*: we will continue to look for more opportunities
   to apply fuzzing to the compilation pipeline. There is likely
   opportunity to (i) build smarter oracles (i.e., better than
   "doesn't crash") for certain parts of codegen, and (ii) stress-test
   certain components by deliberately feeding them "evil" (worst-case)
   input.
   
   - As an exmple of a better oracle, we could fuzz the branch
     peepholing or CFG lowering by checking CFG equivalence across
     these transforms.
     
   - As an example of worst-case input, we could adapt the ABI
     implementation to explicitly clobber caller-save registers; we
     could randomize the regalloc's decisions (scrambling live-range
     priorities and inserting random spills); we could randomize
     basic-block order; we could force "long-form" offsets/references
     on RISC architectures like AArch64 that have limited instruction
     immediate fields.
 
2. *Use code generation*: We will consider ways to use code generation
   to minimize errors in repetitive or tedious code in the
   backends. Note that we had started the new-backend effort in early
   2020 with the thesis that the legalization and recipes system was
   overly complex and hard to work with; a significant part of this
   complexity has to do with the code generation. However, in at least
   the case of instruction-lowering pattern matching, there is reason
   to believe we could build upon Peepmatic (see below) to automate
   the generation of operand-tree matching code.
 
3. *Investigate end-to-end verification possibilities*: We will look
   for ways to check correctness end-to-end, if we can. This is an
   active area of research and not at all certain; but, for example,
   if we could verify (with an SMT solver and formal models of Wasm
   and machine-code semantics) that certain instruction lowerings were
   correct, we would be able to make stronger claims about correct
   compilation. This may be possible only for instructions or basic
   blocks (rather than whole functions) and/or only with certain
   optimizations disabled; but it would be a valuable mode to have.

Performance
-----------

Improving Cranelift's performance remains an important goal. By
*performance* we mean two distinct measures: compile-time performance,
i.e., how fast the compiler can turn CLIF into machine code; and
run-time performance, i.e., how fast the generated machine code
runs. These are often at odds: spending more time running optimization
passes will increase compile time, but could result in substantial
run-time improvements. Because Cranelift is designed to work well as a
JIT backend, when a tradeoff exists, we err on the side of faster
compile time. However, experience with the new backends has shown that
we can sometimes improve both at the same time.

Our ideas for improving performance are, in rough order of
implementation priority:

1. Establish *continuous benchmarking* so that we have objective
   measures of how well we're doing. The two benchmarking RFCs in the
   Bytecode Alliance RFC repo (bytecodealliance/rfcs#3 and
   bytecodealliance/rfcs#4) propose establishing infrastructure
   (servers running some sort of benchmarking bot integrated with
   GitHub) and building a statistically representative and sound
   benchmarking suite, respectively. We hope to have this working
   early in 2021 so that we can become more principled in the way that
   we measure and report improvements from other PRs, and have data to
   make objective design decisions where necessary.
   
2. Continue to *improve codegen quality* incrementally. We have had
   quite good success improving performance on AArch64 and the new
   x86-64 backend by looking at "hot blocks" and fixing the obvious
   problems that stand out; early low-hanging fruit involved things
   such as making good use of addressing modes. There are likely many
   opportunities still remaining to improve our
   lowering. Additionally, the regalloc.rs register allocator has
   improved greatly (and significantly beats the old backend's
   register allocator), but we can continue to look for ways to reduce
   spilling and improve move-coalescing. Rematerialization may be
   fruitful as well.
   
3. Reduce *lowering overhead* by applying *code generation* to ensure
   that operand-tree traversal is done in the most efficient way. The
   work in the past year on Peepmatic has shown that we can generate
   matching automata for trees of operators, and merge individual
   automata (one per matching rule) into a single final automaton that
   factors out the redundant work. While Peepmatic was originally
   designed for CLIF-to-CLIF peephole-optimization transformations, we
   think it should be possible to adapt it for CLIF-to-VCode lowering
   as well. This will require integrating its code-transformation pass
   with the VCode lowering algorithm, and will require extending its
   model somewhat: for example, we will want to factor out
   sub-patterns (e.g., addressing-mode matching) and allow
   sub-patterns to produce strongly typed results (e.g., the `Amode`
   enum) with something like an attribute grammar.
   
   Concretely, doing this would allow us to play with many different
   implementation choices and experiment in a systematic manner: we
   could generate code, do table-driven pattern matching, or some
   combination (trading off code size / icache locality for
   optimization of the matching code); we could control sub-result
   reuse vs. recomputation, and matching depth, in a systematic way;
   we could trim the rule collection down to a smaller, but still
   complete, subset for a smaller and faster (but less optimizing)
   compiler. It would also reduce errors, potentially: there are a
   number of invariants that one must keep in mind when writing a
   backend's lowering code that could be automatically obeyed, or at
   least checked, by the generator. And it would ease the addition of
   new instructions.
   
   One might wonder: the new-backend effort was originally motivated
   by the desire to simplify the web of generated code and separate
   concerns that had to be understood to develop a backend (recipes,
   encodings and legalizations). Are we not just inviting the same
   problems to return if we do this? We argue that the main difference
   comes because of the two-level IR: instructions in VCode are
   machine-specific and strongly typed. Whereas the old design
   represented machine instructions as some sort of refinement of CLIF
   opcodes -- a `load.i64` is encoded as an x86-64 `MOV` with this
   particular operand type and addressing mode because of this
   "encoding" index stuffed into an array, and the details of this
   were scattered throughout the recipe/encoding system -- the new
   design allows the backend developer to directly define machine
   instructions (in the `MachInst` enum) as ordinary Rust data
   structures, and lower into them in a straightforward way with
   open-coded sequences. In contrast, systematically-generated
   matching code is a *good* thing as long as it is clear exactly how
   to add patterns, and as long as there is not too much magic to
   understand.

4. Experiment with *VCode as a machine-independent IR* as well. We
   have discussed at various times what it would mean to instantiate
   `VCode<Inst>` with a machine-independent `Inst` -- perhaps just a
   CLIF `Opcode` with `Reg` arguments/results -- and lower from this
   `VCode<CLIFInst>` to `VCode<x64::Inst>` (for example).
   
   Doing so would carry over the efficiency advantages of a
   linear-array-of-instruction representation, but would carry clear
   downsides if not addressed: (i) VCode is not in SSA form, which
   makes many optimizations more difficult; and (ii) VCode does not
   allow for easy patching, such as removing, inserting, or replacing
   instructions.
   
   We could address (i) by defining a "partial SSA" form: could we
   carry metadata with VCode that, for each register, records either
   "one definition at instruction i" or "many definitions"? A key
   observation here (due to @sunfishcode) is that many SSA-using
   optimizations only do something interesting anyway if a use is not
   defined by a phi-node (i.e., does not have multiple definitions, if
   chased through the phi). It is much cheaper to maintain "single def
   or unknown" partial-SSA than to compute dominance frontiers and
   insert phis (or block params, in our case).
   
   We could address (ii) by designing a "patch" extension to VCode. We
   do this in an ad-hoc way already in regalloc.rs: we build a list of
   insertions, then in one pass, copy instructions into a new list
   interspersed with insertions. Perhaps we could mostly maintain the
   array of instructions but record edits, and occasionally fold these
   edits back into a fresh array.
   
   Taking this further, VCode-as-a-data-structure could provide
   abstractions for passes, and we could fuse adjacent forward or
   backward passes into a single loop. For example, if Peepmatic
   generated VCode in a (reverse-order) stream, we could fuse a
   liveness computation into the same pass.
   
   Finally, if we move to VCode as a central IR, we could abstract
   much of what is needed to do optimizations like GVN,
   constant-folding, and DCE into "instruction attribute" traits; the
   adapted passes could be run both before and after lowering.
   
   Note, again, that we are converging on parts of Cranelift's old
   backend design here (`VCode` rather than CLIF, and lowering rather
   than legalization), except with the key distinction that we
   parameterize the `VCode` on the instruction set; the explicit
   lowering step, and the explicit representation of machine details,
   are the keys to cleaner and more flexible machine backends. (And,
   again, `VCode` carries the data-structure efficiencies and
   potential speedup of partial SSA.)
   
   To be clear, none of this proposed direction is certain to work;
   but we think it seems promising enough that, if we want to push the
   envelope of JIT-targeted compiler efficiency further, we should
   invest effort in finding out.

Security
--------

The most important aspect of security is simply to generate correct
code, which we have covered above. Beyond that, however, there are
various *mitigation strategies* in which a compiler might participate,
and we intend to investigate at least a few.

1. On some architectures, new ISA extensions have been introduced that
   allow the system to do hardware verification or authentication of
   security-critical data such as pointers or control-flow
   targets. For example, on aarch64, the latest ISA revision contains
   pointer-authentication instructions. If we could provide support in
   Cranelift for using these instructions, the various JITs built on
   Cranelift would immediately become more secure.

2. Compiler-based security hardening techniques have existed for some
   time: for example, some techniques require the compiler to
   instrument functions to check for stack or return-pointer
   integrity. Other techniques have the compiler use certain sequences
   to avoid speculation or timing-channel attacks. While we have
   Spectre mitigation on heap accesses (via conditional moves)
   already, we should investigate other options here and ensure that
   we follow best practices in all cases.

Conclusion
----------

We have many ideas for how we might improve Cranelift in 2021! While
we continue to work on all of these, we invite community input; and we
look forward to seeing Cranelift used in new and interesting ways in
the next year.
