# Summary
[summary]: #summary

Introduce Pulley &mdash; a portable, optimizing interpreter &mdash; to Wasmtime.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected -->
<!-- outcome? -->

At present, if you want to use Wasmtime, you are limited to the ISAs that
Cranelift targets.[^or-winch] That means your options are currently `aarch64`,
`riscv64`, `s390x`, and `x86-64`. If you want to use Wasmtime on a 32-bit
architecture, for example, you're currently out of luck. Your only option would
be to add a new Cranelift backend for the ISA, which is a fairly involved task,
especially for someone who isn't already familiar with Cranelift.

[^or-winch]: Or targets where Winch has a backend, but currently this is a
    strict subset of the targets that Cranelift supports.

**Adding a portable interpreter to Wasmtime means that you will be able to take
Wasmtime anywhere that you can compile Rust.** However, you will not have the
near-native execution speeds you can expect from an optimizing Wasm compiler
targeting native code. Therefore, **a secondary goal is execution speed**, to
minimize that performance gap as much as possible. Finally, the introduction of
an interpreter **should not change Wasmtime's public API, nor radically impact
existing Wasmtime internals**. Ideally, this is just a new option on
`wasmtime::Config` next to Cranelift and Winch, and the rest of the public API
otherwise remains as it is today. And ideally, other than the introduction of
the interpreter itself, the rest of Wasmtime's internals are unaffected.

## Non-Goals

It is also worth mentioning some common goals for interpreters that are
explicitly *not* goals for this effort, or at least are not things that we would
sacrifice our main goals for:

* **Being a simple, obviously-correct, reference interpreter:** Wasm already has
  a fantastic reference interpreter written in a simple and direct style that
  closely matches the language specification and can be used, for example, as an
  authoritative differential fuzzing oracle. Furthermore, Cranelift already has
  an interpreter for its intermediate representation for use with testing and
  fuzzing. Taking this implementation approach for Pulley would be redundant and
  would also limit us with regards to our secondary goal of execution speed.

  That said, we will, of course, go to great lengths to ensure safety and
  correctness, but will rely primarily on our existing, extensive fuzzing and
  testing infrastructure for these properties rather than on naivety of
  implementation.

* **Minimizing the duration between first receiving a Wasm binary to the moment
  execution begins:** To achieve the lowest-latency start up, an [in-place
  interpreter][wizard] is required. However, Wasm, as a bytecode format, is not
  conducive to efficient interpretation. It is a stack-based format, which
  allows for compact encoding (useful when transferring Wasm binaries over the
  wire) but also requires more opcodes to perform the same operation than a
  register-based format does. And unfortunately, the expensive part of an
  interpreter is its opcode-switch loop, and doing less work per opcode
  magnifies the cost of the opcode-switch loop. Therefore, to advance our
  secondary goal of execution performance, we will not make Pulley an in-place
  interpreter that prioritizes fast start up.

  However, once the Wasm module is pre-processed, we can expect the same [~5
  microsecond instantiation times that Wasmtime provides for pre-compiled Wasm
  modules][fast-instantiation]. The Wasm can be pre-processed offline and
  ahead-of-time and then sent to the device that will actually execute the Wasm
  module, after which it can enjoy both low-latency instantiation as well as
  improved execution throughput.

* **Dynamically "tiering up" from the interpreter to the compiler:** As a follow
  up to the last non-goal, because this interpreter is not designed for
  minimizing start up times, it is also not intended to be used as an initial
  tier for starting Wasm execution quickly while waiting for our optimizing
  compiler to finish compiling the Wasm in the background, and then dynamically
  switching execution from the interpreter to the compiled code, once it's
  ready.

* **Being the very fastest interpreter in the world:** Finally, we will not aim
  to create the world's very fastest interpreter; doing so involves
  [implementing the interpreter's opcode-switch loop in hand-written
  assembly][luajit-comment]. Writing assembly directly conflicts with our
  primary motivation: portability. That said, you can [achieve codegen that is
  very close to that ideal, hand-written assembly via tail
  calls][musttail]. Unfortunately, the Rust language has not stabilized its
  `become` feature for guaranteed tail calls. Therefore, until Rust stabilizes
  that feature, we are limited to relying on LLVM to recognize our opcode-switch
  loop's code pattern and clean it up into something similar to what it would
  have produced if we had used tail calls. Some initial exploratory
  investigation has shown that LLVM is at least capable of doing that. We can
  also experiment with the combination of macros and cargo features to choose
  between the unstable `become` tail calls or the stable `loop { match opcode {
  .. } }` at compile time without changing our interpreter's source.

# Proposal
[proposal]: #proposal

<!-- The meat of the RFC. Explain the proposal in sufficient detail to support -->
<!-- building consensus around the primary design questions and how they affect -->
<!-- stakeholders. The fine details of a design will be finalized during -->
<!-- implementation review. -->

As Wasm interpreters start trying to improve execution speed, and after they've
already optimized the opcode-switch loop as much as they can, they tend to move
away from Wasm's stack-based bytecode and adopt an internal, register-based
bytecode. When given a Wasm program, they translate it into this internal
bytecode before actually interpreting it. Once this translation step exists,
they start adding peephole optimizations and a simple register allocator to
generate better internal bytecode. They start adding "super-instructions"
(sometimes called "macro ops") that perform the work of multiple Wasm
instructions in a single internal instruction, amortizing the expensive
per-opcode decoding and branch misprediction overheads that plague
interpreters. Although this pipeline still ends with an interpreter loop, the
front half of it looks very much like an optimizing compiler.

The Wasmtime project already leverages an optimizing compiler:
Cranelift. Cranelift has sophisticated mid-end optimizations (the e-graphs
mid-end and its associated rewrites), a robust register allocator (`regalloc2`),
and an excellent DSL for matching multiple input instructions and lowering them
to a single, complex output instruction (ISLE). It has global value numbering,
loop-invariant code motion, redundant load elimination, store-to-load
forwarding, dead-code elimination, and more. What was present in that opimized
interpreter's pipeline, but which Wasmtime and Cranelift are missing, is a
portable internal bytecode that can be interpreted on any architecture that the
Rust compiler can target.

**This RFC proposes that we define a new, low-level bytecode designed for fast
interpretation, add a Cranelift backend to target this low-level bytecode, and
finally implement a portable interpreter for that bytecode.** This lets us
leverage all our existing Cranelift optimizations &mdash; not reimplementing a
subset of them in a custom, ad-hoc, Wasm-to-bytecode translator &mdash; while
still ultimately resulting in an portable, internal bytecode for
interpretation. We reuse shared foundations, minimizing maintenance burden, and
end up with something that is both portable and which should produce
high-quality bytecode for intepretation.

What follows are some general, incomplete, and sometimes-conflicting principles
we should try and follow when designing the bytecode format and its interpreter:

* The bytecode should be simple and fast to decode in software. For example, we
  should avoid overly-complicated bitpacking, and only reach for that kind of
  thing when benchmarks and profiles show it to be of benefit.

* The interpreter should be able to avoid materializing `enum Instruction {
  .. }` values, and instead decode immediates and operands as needed in each
  opcode handler.

* Because we aren't materializing `enum Instruction { .. }` values, we don't
  have to worry about unused padding or one large instruction inflating all
  small instructions, and so we can lean into a variably-sized encoding where
  some instructions are encoded with a single byte and others with many. This
  helps us keep the bytecode compact and cache-efficient.

* We should lean into defining super-instructions. ISLE's pattern matching makes
  finding and taking advantage of opportunities to emit super-instructions easy,
  and the more we do in each turn of the interpreter loop the less we are
  impacted by its overhead.

* We should not define the bytecode such that multiple branches are required to
  handle a single instruction. For example, we should *not* copy how many of
  Cranelift's `MachInst`s are defined where there are nested enum types:

  ```rust
  enum Inst {
      AluOp {
          opcode: AluOpcode,
          ..
      },
      Load { amode: AddrMode, .. },
      ..
  }

  enum AluOpcode { Add, Sub, And, .. }

  enum AddrMode {
      RegOffset { base: Reg, offset: i32 },
      RegShifted { base: Reg, index: Reg, shift: u8 },
      ..
  }
  ```

  This would require branching on an `Inst` to discover that it is an
  `Inst::AluOp` and then branching again on the `AluOpcode` variant to find an
  `AluOpcode::Add`, or similarly branching from an unknown `Inst` to an
  `Inst::Load` and then again from an unknown `AddrMode` to an
  `AddrMode::RegOffset`. Branches are expensive in interpreters since many Wasm
  program locations map to the same native code location inside the core of the
  interpreter, obfuscating patterns from the branch predictor.

  Instead we should do the moral equivalent of the following:

  ```rust
  enum Inst {
      AluAdd { .. },
      AluSub { .. },
      AluAnd { .. },
      LoadRegOffset { base: Reg, offset: i32 },
      LoadRegShifted { base: Reg, index: Reg, shift: u8 },
      ..
  }
  ```

  With this approach, each opcode handler is branch-free and the only branches
  are from an unknown opcode to its handler.

* We should structure the innermost interpreter opcode-switch loop such that we
  can coerce LLVM to produce code like the following pseudo-asm for each opcode
  handler:

  ```python
  handler:
      # Upon entry, the PC is pointing at this handler's opcode.
      #
      # First, decode immediates and operands, advancing the interpreter's PC as
      # we do so.
      operand = load pc+1
      immediate = load pc+2

      # Next, perform the actual operation.
      do_opcode operand, immediate

      # Finally, advance the pc, read the next opcode, and branch to its handler.
      pc = add pc, 3
      next_opcode = load pc
      next_handler = handler_table[next_opcode]
      jump next_handler
  ```

  This results in a minimal number of branches and gives the branch predictor a
  tiny bit more context to make its predictions with, since each handler has its
  own copy of the decode-next-opcode-and-jump-to-the-next-handler sequence.

  We should ideally avoid each handler branching back to the top of a loop which
  then branches again on the PC's current opcode, as this introduces multiple
  branches and obfuscates patterns from the branch predictor.

  Of course, this is going to be on a best-effort basis, largely relying on
  playing nice with LLVM's optimizer, since we aren't writing assembly by hand
  and can't use tail calls yet.

* We should implement a disassembler for the bytecode format so that we can
  integrate the new backend with Cranelift's existing filetests. Ideally this
  disassembler is automatically generated from the bytecode definition.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- What other designs have been considered and what is the rationale for chosing -->
<!-- the one proposed? -->

* Instead of creating our own bytecode format, we could reuse an existing
  format, or even a interpret a real ISA. SpiderMonkey, for example, has an
  ARM32 interpreter that it uses for testing. However, if we do not define our
  own bytecode, then we lose flexibility. For example, we cannot define our own
  super-instructions for common sequences of operations that we recognize. Nor
  can we define the bytecode's encoding and make sure that the overhead of
  decoding instructions in software is as low as possible.

* We could lean into adding Winch backends for new ISAs, instead of adding an
  interpreter, for our portability story as a simpler alternative to a new
  Cranelift backend. However, Winch and Cranelift share a fair amount of code
  for e.g. encoding machine instructions, and it isn't clear that adding a new
  Winch backend is actually any easier than adding a new Cranelift backend. It
  is also still work that will need to be done on a per-target basis. In
  contrast, adding an interpreter gives us support for everything that `rustc`
  can target with a fixed amount of effort, and without any new per-target
  implementation costs on our end.

* Instead of using a side table mapping each opcode to its associated handler,
  we could have the handler's function pointer itself be the opcode, jump
  directly to the next instruction's opcode, avoiding an indirection. However,
  this inflates the bytecode size (an opcode is a pointer-sized value instead of
  a single byte, possibly longer for rarer operations due to a variably-sized
  encoding) which adds additional cache pressure. Additionally, it makes
  serializing the bytecode for use in another process difficult, since the
  opcode changes with ASLR. This latter hurdle is surmountable, we could define
  relocations for each instruction in the bytecode, but this would add
  complexity and slow down start up times, since many relocations would need to
  be resolved when loading a bytecode module. All those relocations would force
  the whole module to be mapped into memory as well.

  This is the approach that JavaScriptCore's Low-Level Interpreter (LLInt)
  previously took, but [they ultimately abandoned it][llint-new-bytecode] when
  they started caching bytecode to disk across sessions and desired a
  more-compact bytecode format.

# Open questions
[open-questions]: #open-questions

<!-- - What parts of the design do you expect to resolve through the RFC process -->
<!--   before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through implementation after -->
<!--   the RFC is accepted? -->

* The exact bits of the bytecode, what super-instructions we ultimately define,
  and the structure of the interpreter all have a bunch of open questions that
  can really only be resolved through implementation and experimentation. I
  expect that we will handle these as they come, and don't think we need to get
  into their specifics before this RFC is merged.

* What is "Pulley" a backronym for? The best I can come up with is "Portable,
  Universal, Low-Level Execution strategY".[^y] That "y" really makes things
  difficult...

  This should definitely be resolved before this RFC merges.

[^y]: Thanks to Chris Fallin for the "strategY" suggestion.

[wizard]: https://arxiv.org/abs/2205.01183
[fast-instantiation]: https://bytecodealliance.org/articles/wasmtime-10-performance#wasm-module-instantiation
[luajit-comment]: https://www.reddit.com/r/programming/comments/badl2/luajit_2_beta_3_is_out_support_both_x32_x64/c0lrus0/
[musttail]: https://blog.reverberate.org/2021/04/21/musttail-efficient-interpreters.html
[llint-new-bytecode]: https://webkit.org/blog/9329/a-new-bytecode-format-for-javascriptcore/
