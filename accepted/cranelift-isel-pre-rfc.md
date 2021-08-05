# Summary

There is a need to improve the mechanisms for describing instruction
lowering logic in Cranelift backends.

The Cranelift compiler backend design was recently (in 2020)
overhauled to simplify its operation and remove the
difficult-to-understand legalization/encoding/recipe-based system. The
new design instead allows for straightforward lowering: the backend
does a `match` over opcodes, and emits some machine-specific
instructions. This has worked well enough, and has allowed for the
proliferation of multiple new backends (new x64, aarch64, s390x,
partial arm32) and a wide range of supported Wasm features (e.g.,
SIMD), all in a more theoretically flexible instruction selector
framework (e.g., allowing backends to merge loads, complex address
generation, and extends/shifts into other operations).

However, as the new backends increase in complexity, there is a
crossover point at which the handwritten pattern-matching itself has
become unwieldy, and a DSL of *some* sort would likely reduce
repetition, avoid errors, and allow for more flexibility in
refactoring.

This pre-RFC explores the design space for "instruction selector
DSLs", and suggests a list of requirements that should be met by any
system which would take over the new codegen backend logic.

The intent of this pre-RFC is to achieve consensus on a set of
requirements (possibly those presented here, or others gathered from
contributors) so that we can design an appropriate DSL, and to pose
discussion questions that will start to direct us toward the
appropriate design. It is, as much as possible, unopinionated: we will
try to summarize pros and cons where design tradeoffs are
discussed. The one exception is that it is slightly biased in the
direction of "we need a DSL" -- otherwise, why discuss this? -- but
this too is a question that is certainly open for discussion.

# Background and Motivation

## Approaches to Instruction Selection

Instruction selection -- the problem of translating a program in some
intermediate representation (IR) into instructions for some CPU
architecture -- is a well-structured problem that lends itself to
approaches that use generated code (sometimes known as the
"meta-compiler" or "compiler-compiler" approach).  This is because, at
some basic level, it is a *pattern-matching* problem, and the code
that performs the translation is matching patterns in the input and
generating certain combinations of instructions in the output for each
pattern.

There are competing approaches to implement such a pattern-matching
system. On the one hand, any framework or system that abstracts the
common logic of pattern matching away from the patterns themselves
will add to complexity. In the initial design for Cranelift's backend
framework (and its first x86-specific lowering rules), this complexity
became overwhelming, and presented problems for engineers wishing to
improve existing backends or contribute new ones [1]. Such a system,
if mismatched to the characteristics of the problem at hand, can also
contribute to inefficiency: for example, if it works by applying rules
incrementally until no more rules apply, there may be unnecessary
overhead in dynamically scheduling rules, and in constructing
intermediate states in memory, when the final translation for a
particular input pattern could be known statically.

On the other hand, one can implement a compiler backend that
translates the input to output instructions in one step with
hand-written code. This is the answer that we arrived at in
Cranelift's new backend design. By going to the other extreme and
avoiding any significant pattern-matching-specific framework, it
achieves simplicity, approachability, and efficiency (in the ideal
case).

[1]: https://github.com/bytecodealliance/wasmtime/issues/1141

This technique is extremely approachable when backends are simple. For
example, the toplevel `match` on opcode and arm for an `add`
instruction might appear as follows, if the backend always lowers an
"add" operation to a single add instruction with no other
operation-combining or optimizations (slightly simplified but
representative):

```rust
fn lower_inst(ctx: &mut Ctx, insn: Insn, /* ... */) {

    let op = ctx.data(insn).opcode();
    match op {
        // ...

        Opcode::Iadd => {
            let rn = put_input_in_reg(ctx, insn, 0);
            let rm = put_input_in_reg(ctx, insn, 1);
            let rd = get_output_reg(ctx, insn, 0);

            ctx.emit(Inst::AluRRR {
                op: Opcode::Add64,
                rn,
                rm,
                rd,
            });
        }
    }
}
```

Adding support for a new IR-level opcode then has relatively little
overhead: it requires only a new match arm and (if needed for the
lowering) new machine instructions in the `Inst` enum.

## Matching Combinations of Operators

The next natural step in this hand-written style is to perform deeper
pattern matching and generate better code when possible, perhaps
matching on, and combining, more than one operation in the input
program into one instruction in the output. Deeper pattern matching is
usually written via ordinary conditionals, forming a decision tree
(or, actually, DAG) of sorts. Decisions are made depending on further
properties of a portion of the input: an opcode, an immediate, a
branch condition, etc. Sometimes the decision is an N-way choice:
match on an input type and generate one of N possible instructions,
for example. In other cases, the decision is binary: e.g., if an
immediate is small enough to fit in a certain field, then generate an
immediate-form instruction, otherwise fall back to general case.

For a simple example of this deeper matching, consider the addition of
`MADD` (multiply-add) support to the above add-op lowering code:

```rust
    match op {
        // ...

        Opcode::Iadd => {
            if let Some((mul_insn, 0)) = maybe_input_insn(ctx, 0, Opcode::Imul) {
                let rn = put_input_in_reg(ctx, mul_insn, 0);
                let rm = put_input_in_reg(ctx, mul_insn, 1);
                let ra = put_input_in_reg(ctx, insn, 1);
                ctx.emit(Inst::AluRRRR { op: Opcode::MAdd64, ... });
            } else if let Some((mul_insn, 1)) = maybe_input_insn(ctx, 1, Opcode::Imul) {
                // ...
            } else {
                // normal Add as above.
            }
        }
    }
```

This is "optimal" in the sense that (i) it checks further up the tree
of operands only if we have matched the pattern so far, and (ii) it
shares work to find the "prefix" of the match (the Add at the
root). With care, more complex patterns can also be combined in this
way, and this has been done successfully in the backends so far.

## Downsides as Complexity Increases

The major downsides of this approach, however, have become more
prominent as the backends grow more complex. The hand-written
simplicity that allowed us to achieve efficient lowering (direct
translation rather than iterative application of in-place edits) and
that made the backends approachable for contributors (to great
success!) has become bogged down in layers of nested, manual
pattern-matching.

There are three major disadvantages. The first is that it is simply
tedious, and this discourages work on more optimal code generation in
specific cases where better lowerings exist. In contrast, a "pattern
language" that could express a new lowering rule in a few lines would
allow for much quicker, and more thorough, optimization of the
backends by domain experts. This is especially relevant for SIMD and
other arithmetic opcodes where complex sequences must be emitted to
obtain the correct semantics in various cases.

The second disadvantage is that hand-written code is difficult to
refactor or migrate across API or backend design changes. For example,
as part of the regalloc2 effort, an attempt was made at first to
migrate enough of the Cranelift backend(s) to (i) generate SSA code
(i.e., do not reuse virtual registers) and (ii) use slightly different
regalloc APIs. This diff grew to 4k lines at ~1/3rd complete, and
involved essentially re-authoring all of the delicate lowering
patterns, with a high chance of introducing a novel bug. In contrast,
code that is generated can be adapted to use new APIs or frameworks by
adjusting the code generator itself, which is usually much easier (the
changes are made only in one place).  More importantly still, the
lowering patterns themselves are not modified, so the semantics (and
correctness) remain exactly the same. Especially as we look toward
further optimizations in our compiler pipeline, this flexibility will
be very important.

The final disadvantage of hand-written pattern matching is that it
becomes increasingly error-prone as more matching rules and
optimizations interact. Two example bugs suffice to illustrate the
problem. First, the security bug CVE-2021-32629 arose out of a case in
which we recognized and elided an unnecessary zero-extend, when the
proper (extended) type was not exposed to the register allocator. A
system that reasons in a principled way about operators with typed
inputs and outputs might not have allowed us to express this
bug. Second, the bug fixed by PR #2576 [2] was caused by an
over-aggressive merge of a load into a compare op that was generated
more than once; again, a system that enforces invariants in the way
that instructions are matched and merged would likely have caught
this.

[2]: https://github.com/bytecodealliance/wasmtime/pull/2576

## The Renewed Need for a Principled Approach

For all of the above reasons, as our machine backends evolve to become
more feature-complete and optimized, and as we have now settled on a
reasonable single-pass lowering design, it is again time to consider
developing a principled approach by which we can express instruction
lowering patterns in some sort of DSL.

Why now, and how will it be different than the
legalization/encoding/recipe system? There are (at least) two reasons:

1. Intrinsic complexity is increasing. We now support more
   instructions (SIMD), and we have more fine-tuned instruction
   lowerings (the old backend could never, e.g., combine address
   generation expressions into complex addressing modes, or make use
   of builtin extend/shift capabilities of instructions).
   
2. We understand the required design better now. The earlier
   legalization-based system was tied to an edit-in-place approach,
   and also to the idea that the machine lowering can simply be
   represented as an overlay of "encodings" on the IR; as discussed in
   [3], we now understand that a two-level IR fits this problem space
   better, and that a streaming one-pass approach to lowering is more
   efficient. It took a descent into simple hand-written code to reach
   this point, and to allow us to bring up three new machine backends.
   But now, since the design has now settled, it makes sense to
   re-abstract into a mechanized approach.
   
[3]: https://cfallin.org/blog/2020/09/18/cranelift-isel-1/

# Requirements for a DSL

Now that we have established the need for some sort of DSL, let's
examine what requirements are imposed by the problem at hand.

## Requirement 1: First-Class Destructuring/Matching

The most important task of instruction-selection DSL is matching on,
and destructuring (binding variables to components of), the input IR.

Thus, the DSL must make these operations as natural and easy to
express as possible. Even a little bit of friction -- say, manually
calling a method, or manually extracting matched components, for each
level of a pattern match -- will discourage efforts to refactor and
optimize lowering rules.

We should strive for the simplicity of other pattern-matching DSLs
that have been used to specify instruction selection patterns. For
example, the Go compiler uses a DSL to generate lowering code from
S-expression patterns such as [4]:

```
// Lowering arithmetic
(Add(64|32|16|8) ...) => (ADD(Q|L|L|L) ...)

// combine add/shift into LEAQ/LEAL
(ADD(L|Q) x (SHL(L|Q)const [3] y)) => (LEA(L|Q)8 x y)

// Merge load and op
((ADD|SUB|AND|OR|XOR)Q x l:(MOVQload [off] {sym} ptr mem)) &&
    canMergeLoadClobber(v, l, x) &&
    clobber(l) =>
((ADD|SUB|AND|OR|XOR)Qload x [off] {sym} ptr mem)
```

[4]: https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/gen/AMD64.rules

Likewise, in LLVM's TableGen-based machine specifications, one can
find expression-tree matchers such as [5]:

```
// A read-modify-write negate: matches (store (ineg (loadi64 ...))) tree.

def NEG64m : RI<0xF7, MRM3m, (outs), (ins i64mem:$dst), "neg{q}\t$dst",
                [(store (ineg (loadi64 addr:$dst)), addr:$dst),
                 (implicit EFLAGS)]>,
                Requires<[In64BitMode]>;
```

[5]: https://github.com/llvm-mirror/llvm/blob/master/lib/Target/X86/X86InstrArithmetic.td

Similarly, in our own tree, Peepmatic can express instruction rewrites
with a straightforward rule syntax that pairs a pattern (the left-hand
side) with a replacement (the right-hand side):

```
(=> (bor $x (bor $x $y))
    (bor $x $y))
```

which could in principle be extended to machine-specific instruction
opcodes.

Whatever DSL we adopt or design should allow one- or two-line patterns
to be written similarly to these whenever a new lowering for a pattern
of IR instructions must be added.

## Requirement 2: Transition from Hand-Written Code

A key fact to consider is that *our machine backends already exist*,
and they must continue working as we implement and make use of
whatever DSL we define. Furthermore, we should do as much as we can to
avoid large atomic switchover PRs. It is much easier to (i) make
progress and (ii) verify correctness as we go if we can migrate small
bits of backend logic at a time.

There are two ways in which DSL-written and hand-written code can
interact:

- "Horizontally", between different instructions. For example, we
  might start to use a DSL to define arithmetic/SIMD instructions, and
  optimize SIMD lowerings, while retaining the existing (working,
  verified, hand-optimized) lowering for all other CLIF
  instructions. Or we might want to prove out the DSL by fully
  implementing all of the existing pattern-matching cases for just one
  instruction. It should be possible to do this while deferring to
  existing handwritten code for all not-handled cases.

  This is the simplest form of co-existence and would allow a
  migration strategy by which we implement the entire lowering logic
  for one or more Cranelift opcodes at a time, eventually covering the
  whole CLIF opcode space.

- "Vertically", within the logic to lower a single CLIF
  instruction. For example, we might use the DSL to define lowering
  patterns for arithmetic instructions and combinations of arithmetic
  instructions, but defer to existing handwritten code that looks for
  loads from memory that can be merged into the arithmetic
  instructions and computes the best addressing mode with more complex
  logic. Or, going the other way, we might try to systematize the
  addressing-mode lowering into a set of rules, and use it from
  existing handwritten code that lowers loads/stores.
  
  This implies a sort of "FFI" by which the DSL code can call out to
  Rust, and perhaps the Rust code can call back into (sub)rules in the
  DSL.
  
  This is a more challenging form of interoperation of old and new
  (though it may fall more naturally out of some approaches than
  others). If we can achieve it, it would allow us to (i) continue to
  make use of shared code, such as code that computes addressing
  modes, various forms of immediates, etc., when writing new lowerings
  in the DSL, and (ii) have more flexibility with certain complex
  lowering logic. The main advantage can thus be described as
  "flexibility".
  
  The main downside of "vertical" interop is that, by providing more
  flexibility, it can make analyzability more difficult. For example,
  if we aim to verify the lowerings, we need to annotate what these
  calls to outside code are doing for the sake of the verification
  tool.
  
  This is reminiscent of "unsafe" code in Rust: it allows one to build
  axiomatic building blocks with flexibility, but it requires one to
  carefully define *what* the building blocks do.

## Requirement 3: Generate "Optimal" Matching Code

Despite a preference for simplicity, the DSL should also work to
achieve *performant* pattern-matching, even when this means that the
generated code works differently than a naïve interpretation of the
semantics of the pattern-matching might suggest. As a simple example,
when a decision rests on an opcode or other parameter and there are N
valid options, we should always do an N-way dispatch with a `match`
rather than a case-by-case linear-time search.

In general, performant lowering should avoid redundant work. This
likely implies factoring out and sharing the results of subtree
matching when multiple instruction-lowering cases would use it, rather
than re-processing the subtree at each query. The system could solve
this either by careful construction and ordering of matching code
(statically) or by memoization (dynamically), or some combination
thereof.

## Requirement 4: Flexible / Minimal Coupling to Cranelift Design

Though not necessary for the initial use-case, a DSL that we develop
*should* be as generic as possible with respect to Cranelift-specific
details.

At the most basic level, one could define a common set of abstractions
that the instruction-lowering system expects to find at its boundaries
-- a set of traits to implement for the input and output IRs, for
example.

An even more generalized design point would allow the concepts of
"instruction" and "operand" to be defined entirely within the DSL
itself. This flexibility comes with an inherent tradeoff: the less
that the DSL compiler knows about the domain, the fewer
domain-specific optimizations that can be applied. However, it might
be the case that all relevant optimizations are actually tied only to
lower-level concepts in the metalanguage, like pattern-matching.

Minimal coupling also allows for use of the lowering rules in
different contexts: for example, it would likely help with
verification efforts (below), and makes testing/fuzzing much more
practical with a smaller testing harness.

## Requirement 5: Ability to Encode Lowering Invariants

While lowering code, there are a number of rules that one must
carefully observe in order to maintain correctness. A simple program
composed only of pure arithmetic operators poses little worry: one can
always duplicate an operator, or move it elsewhere in the program, as
long as the dataflow remains intact. In contrast, (i) loads and
stores, (ii) operators that can trap (such as divides), and (iii)
control flow impose hard limits on which IR-level operators can be
merged together, and which can be hoisted or sunk in the program.

For an example of a somewhat subtle condition, consider the rules that
govern whether a load can be merged into another operator [7]:

```
    /// ...
    /// [This function returns:]
    /// ...
    ///
    /// - An instruction, given that it is effect-free or able to sink its
    ///   effect to the current instruction being lowered, and given it has only
    ///   one output, and if effect-ful, given that this is the only use;
    ///
    /// ...
    ///
    /// If the backend merges the effect of a side-effecting instruction, it
    /// must call `sink_inst()`. When this is called, it indicates that the
    /// effect has been sunk to the current scan location. The sunk
    /// instruction's result(s) must have *no* uses remaining, because it will
    /// not be codegen'd (it has been integrated into the current instruction).
```

[7]: https://github.com/bytecodealliance/wasmtime/blob/9419d635c68f79d12471353e28b60cc8da010dad/cranelift/codegen/src/machinst/lower.rs#L110-L125

The above-cited compare-load merge bug (PR #2576) is a good example of
a bug that can result from a non-obvious violation of these
conditions. There are other cases where we discovered a potential
issue during code review. We have taken care to try to design the
`LowerCtx` API to avoid as many invariant-violations as possible
(e.g., the `Writable<Reg>` type that encodes whether one can write to
a given virtual register), but nevertheless, it remains an area in
which we have to be vigilant.

If we eventually express lowering logic in a DSL, then, we should
ensure that the DSL allows us to encode such restrictions, and
enforces them. For example, we might want the DSL to provide two
different ways to refer to an instruction input: one when we are the
only user, and one when we are not. Then the former might be required
in order to sink a side-effect. Ideally, such restrictions could be
expressed in some sort of DSL-internal type system, so that we do not
have to hardcode lowering rules into the DSL design itself.

## Requirement 6: Helps to Advance Verification Efforts

A top-level goal of the Cranelift project is to actively encourage
verification efforts. As two examples, (i)
[VeriWasm](https://github.com/plsyssec/veriwasm/) statically proves
heap isolation invariants in Cranelift-compiled WebAssembly code, and
(ii) the [regalloc
checker](https://github.com/bytecodealliance/regalloc.rs/blob/main/lib/src/checker.rs)
static proves the correctness of a particular register-allocation
run. More such efforts are expected in the future, with additional
scope.

In order to facilitate verification of instruction selection, the DSL
with which we express the instruction lowering logic should allow for
formal reasoning about the lowering, and correspondence between input
and output. For example, if we were to eventually have a formal
semantics for CLIF, and for the machine code of some architecture, we
should be able to translate the lowering rules in the DSL into
equivalences that tie the two semantics together.

There are two levels of extraction that we might expect to achieve,
with varying levels of difficulty. The first we could consider to be
equivalent to the regalloc checker: the instruction selector generated
from the DSL could produce metadata that would allow us to associate
input and output, and prove *for a specific compilation* that the
result is correct. This is likely much easier (it is very similar to
debuginfo), and is an approach that, e.g., has been recently explored
in research [7]. It also provides an end-to-end guarantee, if done
right: the metadata could tie Wasm bytecode-level semantics to machine
code-level semantics, covering all stages of the compilation
pipeline. If we decide on such a verification approach, then
propagation of this input-to-output correspondence is enough.

The second level of verification is a static "for all inputs"
verification, and requires extraction of all lowering rules in a
symbolic form. This sort of verification is much harder in general,
especially when instruction selection involves code motion (hoisting
or sinking), control flow, or other side-effects; though for the case
of straightline expression-oriented code, one can often prove
equivalence with e.g. an SMT problem formulation. Despite the present
difficulties in the state of the art, we should not prematurely rule
out more general backend verification using symbolic techniques, and
as a result, we should ensure that there is a way to perform such
extraction.

[7]: T Kasampalis, D Park, Z Lin, VS Adve, G Roşu. Language-parametric compiler
     validation with application to LLVM. In ASPLOS 2021.
     https://dl.acm.org/doi/abs/10.1145/3445814.3446751

# Open Discussion Questions

## Do we need a DSL to solve this problem?

It is a significant undertaking to (i) either develop or leverage a
DSL, and (ii) migrate our backends to this DSL. The costs include
implementation effort, learning overhead (by contributors who would
use the system), and risk of introducing bugs.
   
On the other hand, there are significant medium-to-long-term benefits:
it would become significantly easier to make contributions, we would
gain more confidence that the backends are correct, and there could be
significant second-order effects (e.g., new contributors encouraged to
join the project) because of these.

## Compatible with Existing Lowering API/Machinery?

Should our DSL generate code that is compatible with the existing
MachInst lowering API, or do we hook into the compiler pipeline at a
different level?
   
There are interesting implications of the first approach. Along with
"gradual migration" (below), designing for compatibility with the
MachInst lowering logic means that control is inverted relative to a
more self-contained instruction rewriting system. It also implies a
single-pass lowering strategy, rather than an iterative application of
rules.

The major advantage of plugging into the MachInst framework is
pragmatic: it exists and it works, and we can continue to use it both
with handwritten and DSL-generated backends. In other words, the DSL
is just a tool that we use to author these backends. In contrast,
moving to a system that comes with its own toplevel driver and
framework would mean yet another backend-framework transition.

On the other hand, the advantage of the second approach is that it
could mean we take another working system as a whole and attach it to
our compiler pipeline. This grants use greater freedom to use another
DSL, and leverage existing work, if other compelling reasons lead us
to do so.

## Gradual Migration?

Is the gradual-migration approach reasonable? How far should we go to
allow for fine-grained, gradual replacement of backend code with
DSL-generated code, vs. parallel implementations and a toplevel "try
the DSL, then fallback to existing backend" driver?

The major advantage of a gradual migration is that it allows us to get
started much more easily, and to navigate tradeoffs in
reimplementation vs. reuse on a case-by-case basis where complex logic
is involved. For example, if we do not allow intermixing of
hand-written and DSL code within a single instruction lowering, then
before basic arithmetic ops can be supported on x86-64, we need to
migrate the entire addressing mode / load-op merging logic. (We could
build the backend to at least recognize cases where this would happen
instead, and fall back to the old backend; but this would result in
duplicated matching and thus lower efficiency.) Similarly, on both
x86-64 and aarch64, basic arithmetic ops require handling a complex
set of "narrow value mode" logic that inserts extend operators where
appropriate. While both of these examples should be expressible in the
DSL, their relevance to even basic instructions implies that a
"horizontal-only" mix of old and new code would require a very large
initial step.

On the other hand, the major advantage of allowing for a more
coarse-grained transition, and not prioritizing fine-grained
interoperability of handwritten and DSL code, is that it grants more
design-space freedom. If we allow ourselves to consider any
instruction selector design, as long as it fits the `LowerCtx` API, it
is possible we could land at a better design point. This is not to say
that such a final design point could not be reached in a more gradual
manner, especially if a DSL is designed to minimize coupling to
Cranelift specifics, of course.

## Term-Rewriting or Imperative Control Flow?

Is an expression-node / term-rewriting-system design appropriate, or a
system with some imperative flow of lowering actions?
   
Many instruction selection frameworks are built around a fundamental
notion of expression nodes. Every operator in the IR is a node that
produces one or more values, the input is a tree or DAG of these
nodes, and the rules specify equivalences between expressions.

On the other hand, instruction selector languages can also be designed
with an "imperative backbone": match on an input, but then perform
some actions, such as allocating temps and emitting
instructions. Typically this design choice would come as a result of a
principle of "generalized design" (the expression model may not be
flexible enough) and/or as a result of existing backend design.

This design axis is often closely coupled with an inversion of
control. Expression-node or term-rewriting systems typically have an
established core rewriting engine that is part of the system; one
provides rules to this engine and invokes it on some input. On the
other hand, instruction selectors built in a more imperative style
typically more directly interoperate with the rest of the compiler
backend, calling into it as needed, and are invoked by an external
driver loop.

### Examples of Expression-Based Designs

There are many examples of expression-based designs. There are
traditional tree-rewriting instruction selectors, which can be based
on various matching algorithms that often make use of specific
metacompilation-time optimizations (e.g. to build optimal lookup
tables or FSMs). Peepmatic, in our tree, is a good example of this
design point, as is the Go compiler DSL cited above.

Another recently studied option is the "e-graph" (equivalence graph)
abstraction, which applies rewrite rules to a compact representation
of all possible equivalent forms of an expression (see e.g. the
[egg](https://github.com/egraphs-good/egg) crate). An e-graph merges
nodes to compress an exponential option space into typically much more
compact form (in a close analogy to how BDDs compress boolean
functions), and the "all equivalent expressions" abstraction is a
powerful one.

### Examples of Imperative Control Flow

There are also instruction selectors based on a more imperative
design. A handwritten instruction selector is sometimes the best
choice for very fast (JIT-compatible) operation. Our new Cranelift
backends operate in an imperative way, such that a lowering function
is invoked on a particular instruction and it can look "up the tree"
as needed, doing its work in a single pass. This design was inspired
by Valgrind's instruction selection, which also achieves good
efficiency in a single pass.

General tools/languages exist to build pattern-matching in a way that
has explicit control flow: most prominently, the Prolog programming
language provides destructuring and backtracking, and its goal-driven
top-down control flow is closely analogous to how our current
handwritten backends work.

### Advantages of Explicit Control Flow

One might wonder why explicit control flow is advantageous, if we are
only concerned with a small set of imperative actions that are related
to producing an output expression. One answer is that it becomes much
more useful when the lowering interface with the rest of the compiler
backend becomes more complex: for example, when we need to track
side-effect coloring and sink instructions' side effects to merge
them.

It also simplifies the DSL compiler and makes it more predictable. For
an example of what is avoided, a term-rewriting system can unroll a
fixpoint loop that repeatedly applies rewrite rules in order to
produce a single-pass lowering implementation, but this static
unrolling is undecidable in the general case unless the terms are
stratified (separated into layers) and rules are restricted only to
descend these strata, avoiding cycles.

As another example, while term-rewriting engines can merge rules to
share subsets of logic between different lowering paths, it is an
expensive exponential search problem to "factor" the combinations of
rules into shared code in the general case. A lower-level DSL with
explicit control flow allows the author to directly factor into
reusable pieces. Optimizing expression-based rule applications into an
imperative final form is generally a very challenging problem.

Finally, explicit control flow and an imperative paradigm fit more
naturally with the target language (Rust) generated by the
metacompiler, and so allow the DSL user to more easily interface with
the rest of the compiler backend. This is particularly relevant if we
decide that we need fine-grained mixing of handwritten and
DSL-generated code.

### Advantages of an Expression-Based System

The advantages of an expression-based system, on the other hand,
mainly derive from the fact that it imposes more regularity onto the
rules. By limiting or disallowing "escape hatches" -- calling into
handwritten helper code, controlling the sequence of rule applications
explicitly -- it provides a stronger specification of sorts that can
be used for other purposes. For example, while it is likely that an
instruction-selector verification effort could ultimately work with
either approach, it is clearer at the moment that a pure
expression-based system, with equivalence rules, can be directly
verified given semantics of input and output languages.^[6] The more
restricted semantics can also be simpler to understand in some sense:
one is just providing equivalences.

[6]: There are additional questions in how backend verification would
     work that require more research, and so it is somewhat hard to give
     a definitive answer to what DSL features would be required for
     verification. There is an argument that a more end-to-end approach
     would be more appropriate, as many bugs occur in the gaps between
     layers rather than directly in the basic lowering rules. If that's
     the case, then verification might best be done probabilistically
     with a fuzzing-like approach, where we drive the compiler with
     inputs and the compiler produces a "witness proof", or
     correspondence between input and output. However, if we wish to
     *statically* verify *the lowering rules themselves*, this task is
     easier when they are expression equivalences.

Thus the choice here comes down to flexibility, and possibly more
immediate feasibility (in terms of interop with existing code, and
ability to translate existing backends), on the one hand, versus
more regularity but possibly greater challenges in adapting to the
existing system on the other hand.

## Existing DSL?

Is there an existing DSL or language that would be appropriate?

There are a few options to seriously consider here.

### Peepmatic

The most immediate option is Peepmatic: it is already in-tree, and is
built to work on CLIF. The major remaining work if we were to adapt
Peepmatic (as an implementation, not just a general approach) would
likely be along several lines:

1. Modifying the engine to work with the new backend framework, which
   requires some careful thought in how to "invert control" and allow
   the lowering logic to drive the instruction selection.

2. Modifying the engine to operate in a single-pass way rather than
   iteratively applying rewrite rules. This requires us to work out
   how to "unroll" chains of rule applications statically, which
   presents some difficulties as discussed above, but could likely be
   done to a large degree.
   
3. Modifying the engine to (meta)compile the matcher down to Rust
   code, rather than interpret match actions, as it currently does.
   
4. Finding ways to fit much of the "sub-instruction" lowering logic
   that currently exists in the new backends into the expression node
   paradigm, for example by defining special address-mode nodes and
   ensuring that these work efficiently (and are never reified into an
   intermediate version of the program).
   
5. Tuning Peepmatic to any differences in the sorts of rewrite rules
   that we will write.  For example, Peepmatic is built to share
   prefixes and suffixes of match-op sequences between many rules, but
   more complex lowering rules may share the *middle* portion with
   specific prefixes and suffixes. (This connects back to the
   "efficient factoring" point above.)
   
The major upside of using Peepmatic is that it has been well-designed,
well-built, well-tested, and is generally a very high quality tool.

### Some Other Compiler's DSL

It is also possible that we could adapt an instruction selector DSL
from another compiler. For example, we could take more direct
inspiration from LLVM's TableGen and write our own TableGen backend
that generates Rust code. This is a "meta-option" to some degree, as
the "write our own TableGen backend" phrase is hiding a lot of work:
we would still actually need to decide on our matching and lowering
algorithms.

### A Programming Language with Appropriate Semantics

We could also adopt a more general programming language, with a
special fit in some way for the task, as a basis in which to write an
instruction selector. Prolog is a good example: it supports
destructuring, backtracking, is very high-level, and a natural fit for
writing sets of patterns. We would need, however, to carefully work
out the compilation strategy.

### Our Own DSL

Finally, we could decide on the necessary semantics and develop a new
small DSL that meets our needs. The author is experimenting with one
such design point (see note below) but it is by no means set; this is
just a fact-finding / research effort to either prove out or disprove
certain ideas.
     
# Next Steps

We should come to a consensus on requirements above; once we agree on
general requirements and design-space tradeoffs, we can discuss the
DSL semantics in more detail.

As a fact-finding side-mission of sorts, I (@cfallin) am currently
prototyping a DSL that is oriented around pattern-matching,
destructuring, and backtracking, and can be seen as a more
general-purpose language in which one can define an instruction
selector. It allows calling into the existing Rust code and
intermixing DSL-generated and handwritten code at a fine
granularity. The open questions are (i) whether this approach allows
for generating "optimal" instruction selectors as well as higher-level
(e.g., instruction pattern-based) approaches, and (ii) how well this
meets verification-related goals. This is by no means settled or
decided upon; I am just as happy to abandon the effort if it turns out
that a different approach meets our needs better.

# Acknowledgments

Thanks to @fitzgen, @sunfishcode, and @tschneidereit for extensive
discussions and guidance that have allowed the thinking above to
develop to this point.
