# Summary

This RFC proposes a new instruction selection DSL for Cranelift, based
on initial discussions begun with a pre-RFC (bytecodealliance/rfcs#13).

This RFC proposes the name "ISLE" (Instruction Selection/Lowering
Expressions), pronounced like the word for a small island (silent
's'), which is conveniently also an anagram of "isel".

This DSL combines aspects of an earlier prototype design that the
author has developed with ideas from term-rewriting systems, and in
particular plans to implement on top of Peepmatic's automata-based
compilation strategy with a new Rust-generating backend.

The basic principles, in terms introduced/framed by the pre-RFC linked
above, are:

- Expression/value-based design with term-rewriting-like semantics.
- Interaction with existing system, including rich query APIs, via
  "virtual nodes" defined by external extractor functions.
- Typed terms.
- Rule application order guided by explicit priorities, but
  semantically undefined (hence flexible).
- Single-pass lowering semantics via unrolling.

We will dive into each of these principles in more detail below.

# Intro: Term-Rewriting System

The basic concept behind the sort of system that this DSL embodies is
that of *term-rewriting*. In brief, this means that the input and
output of the system are trees (or more general graphs) of *terms*
(symbolic nodes with arguments), and the system transforms input to
output by applying a set of *rewrite rules*. Each rule matches on some
*pattern* -- for example, a term of a particular type with an input
that matches some sub-pattern -- and replaces it with a new tree of
terms.

This is a natural abstraction for transforming code. If we define a
term view of the IR on the input side and machine instructions on the
output side, such that a term represents an operator that consumes
values and produces a value, then the rewrite rules just define
*equivalences*. These rules are easier to validate than other sorts of
compiler-backend implementations because "all" one has to do is to
show that the two sides of the equivalence are equal, separately for
each rule (i.e., the validation is modular).

An example of a rewrite rule might be:

```plain

    (rule (Iadd a b)
          (X86Add a b))
```

which means that an `Iadd` "term" with two arguments `a` and `b` (the
"left-hand side" or "pattern" of the rule) can be written to an
`X86Add` term with the same arguments (the "right-hand side" or
"expression"). One could imagine many such patterns for particular
combinations that the target machine supports.

The concept of a rewrite rule, transforming one term into another
equivalent one, provides a *simple, understandable semantics*. The
importance of this cannot be overstated: if the core of the DSL is
complex because its design is intertwined with a particular backend
design or strategy, or other incidental details, then one will not be
able to use it effectively without understanding implementation
details. On the other hand, if we have a clean rewrite-based
semantics, one can easily convince oneself that a new rule is safe to
add as long as it denotes a true equivalence. This is similar to how
"equational reasoning", in functional programming terms (i.e., that
one can always substitute a function invocation with its definition),
makes a program's semantics easier to understand.

The order in which rewrite rules will be applied can be influenced by
user-specified priorities. However, the language definition is careful
to *not* specify which rule must be applied first; priorities are just
heuristics, or hints. This leaves room for the DSL use-case to apply
rules differently: for example, we might test/fuzz a backend by
applying rules in a different order. One should also be free to tweak
priorities to attempt to optimize, without fear of breaking the
semantics.
    
Note that the ordering is only undefined when there are multiple legal
choices according to the types and available rules. In other words,
this does *not* mean that we will arbitrarily steer into a dead-end in
the graph of rewrite steps; an instruction that is lowerable should be
lowerable with any rule ordering heuristic.

# Extractors: Programmable Matching on Virtual Nodes

The first key innovation that this DSL introduces is in how it allows
patterns to *match* on input terms.

A vanilla term-rewriting system operates on trees of nodes and
transforms a single input tree (which is usually reified as an actual
in-memory data structure) into a single output tree. While this is an
easily-understood model, the requirement to define a *single* view of
the input can unnecessarily restrict expressivity. Said another way:
we sometimes want to use terms to represent certain properties of a
value, and it is difficult to arrange the terms into a tree and then
write the pattern-matching rules in a way that is flexible enough to
support multiple types of queries.

## Motivation

For a concrete example, let us assume we a term type representing a
constant integer value

```plain
    (iconst 42)
```

and we are performing instruction selection to lower to a CPU
instruction set that supports multiple different *integer immediate
formats* for its instructions. For example, AArch64 supports a 12-bit
immediate (unshifted or shifted left 12 bits) in some of its
arithmetic instructions, and a special "logic-instruction immediate"
with bit-pattern generation in other instructions. In a conventional
term-rewriting system, with helper functions defined appropriately, we
might have rules like

```plain
    ;; 12-bit immediate form.
    (rule (iconst val)
          (when (is-aarch64-imm12-value val))  ;; Conditional predicate
                                               ;; that evaluates custom logic.
          (aarch64-imm12 val))
    ;; Logical immediate form.
    (rule (iconst val)
          (when (is-aarch64-logical-imm-value val))
          (aarch64-logical-imm val))
```

which turns the `iconst` into an `aarch64-imm12` node and/or an
`aarch64-logical-imm` node when the predicates allow. Then we might
have rules that pattern-match on `aarch64-imm12` or
`aarch64-logical-imm` terms in certain argument positions in certain
instructions:

```plain
    (rule (iadd ra imm@(aarch64-imm12 val))        ;; bind `imm` to subterm
          (aarch64-add-reg-imm ra imm))
    (rule (ior  ra imm@(aarch64-logical-imm val))  ;; bind `imm` to subterm
          (aarch64-or-reg-imm imm))
```

The question then becomes: how do we build a lowering engine that can
successfully use these rules? If we start with the semantics of "apply
any rule that fits", we might have a situation where an immediate
value (say, `1`) can be represented in *multiple* forms, so it might
be nondeterministically rewritten to either of the above terms. This
is fundamentally a *search problem*: we need to know which rewrite
path to take on the immediate term before we see its use.

What we want instead is a goal-directed search of sorts: starting from
the top-level opcode match (`iadd` or `ior`), we try to rewrite into
only the appropriate immediate form. This is what the equivalent
handwritten backend code would do. This avoids the general
graph-search problem, instead allowing a more directed form of
execution.

## Extractor (Left-hand Side) Functions

The key idea that we introduce is that of the "extractor
function". This looks like a term on the left-hand side of a pattern
-- in fact, a rule that uses one will look identical to the above:

```plain
    (=> (iadd ra (aarch64-imm12 val))
        (aarch64-add-reg-imm ra val))
```

The essential difference is that this is not matching on an
already-rewritten `aarch64-imm12` term. Rather, it is *invoking an
extractor* called `aarch64-imm12` that is a sort of programmable match
operator. It receives the value that it is meant to match (here, the
original `iconst` term) and produces either values for its subterms,
or fails to match. In other words, it is exactly the *reverse* of a
normal function application -- for the type

```
    (decl aarch64-imm12 (AArch64Imm12) IConst)
```

the extractor is a (partial) function *from* the `IConst` type (what
would be the "return type" for an ordinary function) to the
`AArch64Imm12` type (an "argument"). More generally, the extractor can
extract one value into multiple values, just as a forward function can
have multiple arguments.

This language concept was largely inspired by the Scala [extractor
object](https://docs.scala-lang.org/tour/extractor-objects.html)
feature, which allows an `unapply` function to be defined on classes
(as the dual of `apply`) to allow for programmable `match` semantics
in a very similar way.

## Equivalence to Base Term-Rewriting Forms

Note that so far, this is actually completely equivalent to a base
term-rewriting system with only tree pattern-matching. The only
distinction is that the extractor makes the attempt to perform a
particular rewrite *explicit*, or *subordinate* to another rule
application. It allows for natural structuring of rewrites through
intermediate steps so that we don't need a more general scheduling
algorithm.

## External Extractor and Constructor Invocations

The next conceptual leap is that the extractor functions can serve as
the *sole* way to access the input to the term-rewriting system if we
allow them to be *externally-defined*. In other words, the system does
not need to have a native data structure or abstraction for a tree of
terms/nodes at the input, with enumerable contents. Rather, it can
have *only* extractor functions that lower to calls to external Rust
functions. Every "match" operation in the left-hand side of a pattern
is a programmable operation that bottoms out in "runtime" code of some
sort, and the DSL compiler does not need to have any
application-specific implementation of match logic. (As an
optimization, we allow constant integer/enum value patterns directly,
but these could be custom extractor functions with no fundamental loss
of expressivity.)

This is a sort of "foreign function interface" or "FFI" for ISLE: it
allows the backend author (or more likely, the author of a common
"prelude" that provides definitions for all machine backends) to build
abstractions in ISLE on top of the rest of the compiler's backend
APIs.

So far we have seen a way to ingest the *input* tree (i.e., the IR),
but not to produce the *output* tree. To handle this in an analogous
way, we define "constructors" that are also defined externally and
invoked from the DSL. This allows slightly more flexibility than a
pure "output is a tree of Rust enums" design would allow: for example,
one can have a `(new-temp ty)` constructor that can be bound to a
variable (see `(let ...)` form below) and used in the output tree,
which in the generated Rust code becomes a call to
e.g. `LowerCtx::alloc_tmp()`. More generally, this means that
constructors return values that are arbitrary, not just the
unevaluated literal term tree; e.g. `new-temp` could return a value of
type `Reg` that is actually a virtual register. So in a sense this
folds the code-editing/generating actions that the backend would
perform with the final rewritten term tree into the production of the
term tree itself.

## Virtual Internal Terms

So far above, we have only discussed matching on the input tree (via
extractors) and producing the output tree (via constructors). However,
if a chain of rules applies in succession, with a rule matching on a
term that was produced by a prior rule application, we ideally should
allow the rules to chain directly without calling out to external
constructors and extractors. Below we describe the "single-pass"
compilation strategy; for now we just note that in addition to the
above mechanisms, we allow for "virtual" terms that are never actually
constructed but only connect rules together.

# Heterogeneous Term Types

A traditional term-rewriting system has one "type": the expression
tree node. However, when performing instruction lowering, there are
often pieces of an instruction, or intermediate knowledge constructed
about a certain subexpression, that *should* have a more specific
type. One could imagine a series of rewrite rules that transform the
address input to a load instruction into an `(Amode ...)` term that
encodes a machine-specific addressing mode; but this is not a general
instruction, and should not be interchangeable with others. At an even
more basic level, a set of lowering rules from a machine-independent
IR to a machine instruction set deals with two disjoint sets of node
kinds (the input and the output). Ideally these would be typed
differently.

To address this need, the proposed DSL assigns a type to every term. A
particular term symbol -- which may denote an extractor function, a
constructor function, or an internal term -- has a static type. This
allows "FFI" signatures to be statically typed as well.

Concretely, the types correspond to Rust enums or primitive types in
the generated Rust code. When a term's type is a Rust enum, there are
implicit extractors and constructors corresponding to that enum's
variants.
  
# Language Design: Types, Rules, and Export/Import Forms

## Types

First, we allow the user to define types. The type system permits two
kinds of types: primitives, which are always (in Rust terminology)
`Copy` integer-like types, and enums, which correspond to Rust enums
with variants that have named, typed fields and are semantically
passed-by-value (though passed by borrows whenever applicable in
generated code).

A type declaration can take two forms: it can define a new type local
to the DSL definitions, used solely for internal terms (those that are
produced by some rules and consumed by others, but never present in
the final output of the system); or it can define a type that is also
present in existing Rust code that will be linked with the generated
code, so we merely refer to it.

A type declaration is then:

```plain

    ;; Enum with two variants.
    ;; Field types can be primitives or other enums.
    ;;
    ;; Note that types must be acyclic: an enum field
    ;; cannot refer to the containing enum or any other enum
    ;; that refers to it. (We may relax this later if needed
    ;; but it requires reasoning about boxing so adds complexity.)

    (type A (enum (B (x u32) (y u32))
                  (C (z u32) (w u32))))
   
   ;; Same, but we do not emit a Rust `enum B { ... }` for it;
   ;; we assume it has been declared elsewhere, and merely
   ;; use it. Note that variants that are not matched on or
   ;; constructed can be omitted.
   (type B extern (enum ...))
   
   ;; Particular example for `Opcode`: if variant types are declared
   ;; without fields here then they will be matched on and constructed
   ;; in field-less form (e.g. `Opcode::Iadd => { ... }`).
   (type Opcode extern (enum Iadd Imul Isub Iconst ...))
   
   ;; Primitive type. These are presumably included in a prelude
   ;; and should not normally need to be declared by a compiler
   ;; backend author using the DSL.
   (type u32 primitive)
```

Once we have the value types defined, we define *type signatures for
terms*. Every term that is used -- as an internal node, as an
extractor or as a constructor -- needs types for its arguments, and
the whole expression is given a type (analogous to a return-value
type).

```plain

    ;; `Iadd` is a term symbol, and `(Iadd ...)` terms take two arguments
    ;; of type `Value` and produce a result of type `Value`.
    (decl (Iadd Value Value) Value)
    
    (decl (Iconst u32) Value)
```

Now, given a proper type environment for values and terms, we need to
define ways to process them: with extractors and constructors, and
with internal rules. A particular term can *either* have an external
extractor and/or constructor function, or can have an internal rule,
but never both. If a term is to be defined by an extractor and/or
constructor, it must be declared with an `extern` like so:

```plain
    (decl extern (MyTerm u32) u32)
```

## Defining External Extractors and Constructors

First, we define *extractors* and *constructors* for some terms. These
terms are those that represent the external interface to the
instruction selector rules. Extractors, as introduced above, provide a
view onto a virtual term tree at the input.

One can think of an extractor like a "reverse function": it is given
the value of the whole term tree that the extractor is meant to match,
and if it matches successfully, it returns values for each of the
arguments. Extractors are attached to terms as follows:

```plain

    (decl extern (concat u32 u32) u64)
    (extractor concat "unconcat")
```

This indicates to the DSL compiler that the `concat` term can be used
in the left-hand side of a rule (as described below), and when a match
is evaluated, a Rust function `unconcat` will be called. This external
function is assumed to have the signature:

```rust

    fn unconcat<C>(context: &mut C, value: u64) -> Option<(u32, u32)>;
```

If the function returns a `Some((a, b))` value, the match was a
success and the arguments' values are recursively matched by
sub-patterns.

Similarly, a constructor can be attached to a term as follows:

```plain

    (decl extern (concat u32 u32) u64)
    (constructor concat "concat")
```

which implies the existence of a Rust function:

```rust

    fn concat<C>(context: &mut C, val1: u32, val2: u32) -> u64;
```

Note that a constructor, unlike an extractor, cannot fail. More on the
execution model below.

## Defining Rules

The final way to define a term -- and the most common -- is to provide
a set of pattern-matching rules for it. A rule consists of two parts,
the *left-hand side* and the *right-hand side*. The rule specifies
that whenever a term of form that matches the left-hand side exists
and is to be reduced, it can be rewritten as the right-hand side.

Several examples of rule definitions follow:

```plain

    (rule (A x y z) (B x y z))  ;; rewrite (A x y z) to (B x y z)
    
    (rule (A x @ (B 42 _))  ;; match (A (B 42 _)), binding x to the B subterm;
                            ;; the `42` matches a constant value;
                            ;; the `_` matches anything (wildcard).
      (C x))
```

Formally, a rule is defined as `(rule pattern expr)` where the grammar for
the left-hand side `pattern` is:

```plain

    pattern := (extractor pattern*)
             | (internal-rule-term pattern*)
             | constant-integer
             | constant-enum
             | (enum-variant pattern*)
             | variable @ pattern  ;; Match, and also bind this subtree.
             | variable        ;; Bind any subtree (first occurrence) or match
                               ;; same value as variable bound earlier.
             | (and pattern*)  ;; Match all subpatterns.
             | `_`             ;; Wildcard.
```

and the right-hand side `expr` is:

```plain

    expr := (constructor expr*)
          | (internal-rule-term expr*)
          | constant-integer
          | constant-enum
          | (enum-variant expr*)
          | variable
          | (let ((variable expr)*) expr)
```

The evaluation semantics are described in more detail below. Note that
the use of terms defined by other rules, in both patterns and in
expressions, is somewhat subtle and will be discussed further under
"Single-Pass Elaboration" below.
  
## Exporting an Entry Point

Finally, once we have a series of rules that define an instruction
lowering, we must provide an *entry point* by which an external user
can start the rule-matching process.

This may seem a bit odd at first: isn't the concept of an "entry
point" somewhat tied to an imperative view of the world, as opposed to
a list of declarative expression-equivalence rules? The need for this
designation lies in ISLE's evaluation process, which is somewhat
different from a vanilla "apply any applicable rule until fixpoint"
approach. In particular, the matching procedure for any particular
"root term" (at the root of the input tree) can be statically built as
an automaton that combines all patterns rooted with that symbol; and
when we use another internal term in an expression, we can immediately
inline any rules that would apply to *that* term. (More details are
given below.)  So the matching process is a "push" rather than "pull"
process: we invoke generated matching code with a term, and it matches
(via extractor calls) until it produces a final output (via
constructor calls).

Because of this, the matching must be started (the initial "push") by
invoking an entry point corresponding to a particular root term, with
arguments for that term. This is the reason that the ISLE DSL code
needs to export a term that has been defined with rules as an "entry
point". To do so, we write:

```plain

    ;; We have an internal term `LowerInst` that, when wrapping
    ;; an `Inst` term, matches a list of rewrite rules that perform
    ;; the lowering. In other words, we rewrite
    ;;
    ;;   (LowerInst (Iadd ...))
    ;; to
    ;;   (X86Add ...)
    ;; or similar.
    ;;
    ;; This is a little different than a straight rewrite from
    ;; `Iadd` to `X86Add`, but only superficially; it serves to
    ;; give us one term at the root to which we can anchor
    ;; all of our rules.
    
    (decl (LowerInst (Inst) MachineInst)
    
    (rule (LowerInst (Iadd ...) (X86Add ...)))
    
    (export LowerInst "backend_lower")
```

This will generate a callable entry point with a Rust function
signature as follows:

```rust

    fn backend_lower<C>(context: &mut C, inst: &Inst) -> bool;
```

that (i) produces the final output by invoking the declared
constructors with the given `context`, and (ii) returns a boolean
indicating whether the toplevel rule matched (because rules can be
partial; no argument-coverage checking is done).

# Evaluation Semantics

The evaluation semantics of ISLE are carefully defined to (i) ensure
that the effects of rules are simple to understand, (ii) ensure an
efficient compilation of the matching rules is possible, and (iii)
reserve enough freedom to consumers of the DSL that rules can be
flexibly used in many ways.

Evaluation is initiated by a toplevel call into an entry point,
defined as above, and proceeds in two phases: the "pattern-match"
phase and the "expression-build" phase.

When evaluating a pattern, we always know (i) the root term's symbol
and (ii) the value (of some type in ISLE's type system) of the root
term's expression. At the entry point, these are explicitly given (the
entry-point term is fixed, and the values of arguments are given as
arguments to the entry point). Then, as execution progresses and we
match on subterms, we have their root symbols and values as well.

If the root term's symbol is `extern`, and has an extractor, we invoke
the extractor with the given value, receiving the argument values in
return. If the extractor fails the match, we return to the most nested
choice-point (at which we have multiple applicable rules) and try the
next rule.

Otherwise, if the root term is defined by ISLE-internal rules, we
first determine which rules are applicable: these are any with the
root term as the root of their pattern. Record this as a "choice
point" to which we can backtrack. Let us then arbitrarily pick one
rule according to some prioritization scheme. The semantics are
defined such that we try all applicable rules, in some arbitrary
order, and the first that matches wins. If more than one would match,
results are determined only by the prioritization scheme. For each
applicable rule, we try to match the sub-patterns against the known
argument values, recursing as appropriate.

Note that there is a way to compile this recursive-backtracking match
behavior into a set of linearized match operator sequences, where each
operator is an extractor invocation or a constant-value match. This
can then be combined into more efficient code, e.g. by use of an
automaton abstraction. See below for more.

When a rule's pattern (left-hand side) matches, we proceed to
evaluation of the right-hand side. This occurs in the usual
syntax-directed way: we evaluate argument values recursively, then
evaluate constructors with those argument values. Variables are bound
to the values encountered during pattern-matching.

An important invariant during evaluation is that for any given rule,
the pattern-matching phase is fallible, with failure causing the
matching process to progress to the next rule; but the
expression-evaluating phase is *infallible*, and must complete once
started. This is important because constructors (on the right-hand
side) may have side-effects, e.g. allocating temporary registers for
use; we do not want to call into the embedder to produce the final
term until we know we have the correct rule to do so. This strict
phase separation allows for cleaner reasoning about semantics in
general.

In all of the above, we have glossed over the *chaining* of several
rules. What happens when a pattern on the left-hand side refers to an
internal term (produced by other ISLE rules), or when an expression on
the right-hand side produces an internal term?

The key to understanding this behavior is to see that if we inline and
optimize all invoked rules together, we can cancel paired
internal-term expressions that construct the terms and internal-term
patterns that match the terms. Two cases are relevant:

- When during expression evaluation we see a constructor call to an
  internal term, we immediately look up all applicable rules that
  match on that term at the root. We can do a sort of "constant
  propagation" wherein we symbolically match the patterns against the
  constructed internal-term tree. This may statically determine which
  rule we invoke and whose expression we can immediately inline. Or,
  multiple rules may potentially match, and so we need to attempt
  fallible matching within an inner backtracking scope.
  
  Note that though this inlining can result in further internal
  backtracking, it will not cause the calling rule to fail; the rule
  is committed once we reach the right-hand side.
  
  An example of several cases follows:
  
```plain

    (rule (A (SubTerm x)) (B x))
    (rule (A (MyExtractor y)) (C y))
    
    ;; We can inline `A` straightforwardly here, with the `SubTerm`
    ;; introduced and eliminated internally, never exposed to the
    ;; outside world:
    (rule (EntryPoint (D x)) (A (SubTerm x)))
    
    ;; Here, inlining `A` requires us to do a further fallible
    ;; match on the input argument, invoking `MyExtractor` (an external
    ;; extractor) on `x`:
    (rule (EntryPoint x) (A x))
```
  
- When during pattern-matching we see an internal term that does not
  statically resolve, we need to run any defined rules "in reverse".
  An example might serve to illustrate:
  
```plain
    (rule (A x 1) (B x))
    (rule (A x 2) (C x))
    
    ;; The `(B x)` pattern, because `B` is an internal term and not
    ;; an extractor, will search for any rule that produces `B` at the
    ;; root, then substitute the pattern for that rule (here `(A x 1)`)
    ;; at that point in the pattern.
    (rule (EntryPoint (B x)) x)
```

  In essence, one can see an internal term as being equivalent to (i)
  an extractor and (ii) a constructor definition, each of whose bodies
  is composed of logic collected from (i) all rules that produce this
  internal term, and (ii) all rules that match on this internal term,
  respectively. In fact, this is a valid (though possibly inefficient)
  compilation strategy.
  
The above implies that defining a rule with `(rule ...)` may attach an
"internal extractor" to an internal term, if one is produced at the
root on the right-hand side, and/or an "internal constructor" to an
internal term, if one is matched at the root on the left-hand side. If
the left-hand side pattern's *root* symbol is an externally-defined
extractor, then this means that it will only ever be evaluated in then
reverse direction. Similarly, if the right-hand side expression's
*root* symbol is an externally-defined constructor, then this means it
will only ever be evaluated in the forward direction. An example of
the former -- a rule that has an external extractor at the pattern
root root -- follows:
    
```plain

    (decl extern (Extractor ...) ...)
    (extractor Extractor "extractor")
    
    (decl (A ...) ...)
    
    (rule (Extractor a) (A a))
    
    (decl (EntryPoint ...) ...)
    
    ;; This will inline `A` in the pattern-matching phase, running
    ;; "in reverse", bottoming out in a call to `extractor`.
    (rule (EntryPoint (A a)) a)
```
  
# Compilation Strategy

In order to compile ISLE to Rust code, we perform several steps.

## Single-Pass Elaboration

A traditional term-rewriting system typically works in an iterative
fashion: rewrite rules are applied, editing the representation
(e.g. expression tree) in memory, until no more rules apply. This
*dynamic scheduling* of rewrite rules is costly, as is the actual
reification of the intermediate states. In contrast, we expect this
DSL to be compiled into a single-pass transform, and we add certain
restrictions to rules to enable this.

The basic strategy is to "inline" the rule-matching that would apply
to the newly constructed value on the right-hand side of a rule. In
other words, if a rule transforms `(A x y)` to `(B x y)` and there is
another rule that matches on `(B x y)`, we generate code that chains
these two transforms together, effectively inlining the `(A x y)`
extractor on the left-hand side of the latter rule. We call this
inlining "elaboration", in analogy to a similar flattening step in
other hierarchical-design DSLs (in particular, hardware-description
languages).

To ensure that elaboration terminates, we disallow recursion in
rewrite rules. This implies a stratification of term constructors:
terms of one kind `A` can rewrite to `B`, but `B` cannot rewrite back
to `A` (or to another `B` expression). The backends will break
recursion that would otherwise occur in an isel context via external
constructors and references/names: e.g., "get input in reg" does not
immediately recursively lower, but just marks that instruction as used
and it will be lowered later in the scan.

It is an open question whether limited recursion could be allowed
either by (i) statically bounding the unrolling depth, or (ii)
requiring some sort of substructural recursion, in the same way that
some "total" functional languages used in theorem-proving (e.g. Coq)
ensure termination.

## Linearization into Match Operators

Once we have completely elaborated a rule, we lower its actions into a
straight-line sequence of (i) match operators, in the matching phase,
and (ii) constructor calls, in the evaluation phase.

Match operators consist of calls to external extractors, destructuring
of native Rust enum types, matches on constant values, and matches on
already-bound variable values. Each match operator may fail (causing
evaluation to proceed to another rule), or may succeed and optionally
bind variables.

Once we have a linearized sequence of operators for a given root term,
we can combine these sequences into an automaton for that term. We
perform this lowering for the entry-point (and it will naturally
inline other rules as needed during elaboration). The combination of
linearized operator sequences into an automaton will rely on
Peepmatic's existing machinery.

## Prioritization of Rules and Extractors

We have so far not discussed how the matching phase chooses which rule
to try first. This ordering is decided by a set of priorities assigned
to (i) rules and (ii) extractors.

Any rule for an internal term may have a priority attached, as follows:

```plain

    (rule (prio 10) (A x) (B x))
```

Likewise, any extractor may have a priority attached, as follows:

```plain

    (extractor (prio 10) MyTerm "my_term")
```

When dispatching on a certain root term during pattern-matching, we
collect all priorities and sort in descending order. The default
priority is zero for all term matchers and -100 for wildcards (this
will naturally place more specific rules above more general ones in
priority order). If two or more options have the same priority, we
decide arbitrarily (but deterministically for a given version of the
ISLE compiler, to avoid non-reproducible build issues).

# Discussion: Comparison with Hand-Written Backends

We discussed in more depth in bytecodealliance/rfcs#13 how the DSL
approach in general compares to hand-written instruction lowering
code. A few points are notable with specific reference to the design
described in this RFC, however.

First, the handwritten code in our existing `MachInst`-style backends
is fundamentally an imperative design: the machine-independent driver
invokes a backend function once per CLIF instruction to lower, and
this driver in turn queries the IR and eventually performs actions to
build the lowered code (allocate temps and emit machine instructions,
mainly). The earlier pre-RFC contrasted this approach with a
declarative DSL one, in which the execution semantics of the DSL were
not prescribed. Note, however, that this DSL design still takes
efficient compilation *into account*, even if it does not prescribe
the actual execution sequence. Specifically, the design of the
dispatch mechanism, in which the root term (determined first by the
entry point, then by matches on or constructions of internal terms)
statically dispatches to a set of rules that can be combined into an
efficient automaton, it enables a "single-pass" approach that is
distinct from most iterative term-rewriting systems.

Second, the DSL encourages a mode of thinking about instructions (both
in the IR and at the machine level, in VCode) that is more
value-oriented. The `LowerCtx` API is designed to be a sort of general
query API, and instruction output registers are just attributes of an
instruction like any other. Similarly, allocating temporary registers
and emitting instructions are just imperative actions taken by
lowering code; there is no notion of the emitted instructions being
"equal" to values on the input side. In contrast, the ISLE DSL design
privileges the notion of an instruction's "value". This makes the
notation more natural when expressing algebraic-style reductions, and
is consistent with many other instruction-selection systems. (As a
particular implementation note, multiple-output instructions should
work within this framework as well because one can treat the N-output
tuple as a value with constructors and extractors as necessary.)
However, ISLE does not go as far as an instruction-selector framework
that is explicitly aware of, e.g., SSA values: rather, that is up to
the constructors that have been defined. The ISLE system itself only
understands terms and trees of terms; this just happens to be a
natural abstraction with which to represent expression trees.

# Discussion: Future Implications for Verification

Though we have not yet worked out all the details, we are confident
that the translation of rules expressed in the ISLE DSL into some
machine-readable form for formal verification efforts should be
possible. This is primarily because of the "equivalent-value"
semantics that are inherent in a term-rewriting system. The
denotational value of a term is the symbolic or concrete value
produced by the instruction it represents (depending on the
interpretation); so "all" we have to do is to write, e.g.,
pre/post-conditions for some SMT-solver or theorem-prover that
describe the semantics of instruction terms on either side of the
translation.

Note that while externally-defined extractors and constructors at
first glance may appear to make this more difficult, because they
define an "FFI" into imperative code, in actuality we can just treat
them as axiomatic building blocks. In essence, they are the pieces
that define the input and output instruction sets, and so are tied to
the definitions of the formal semantics for these two program
representations; we would start by formally describing the program
value represented by the result of an extractor on an instruction,
and/or the preconditions it implies, then carry through the
implications of this to intermediate internal terms and finally to the
tree of constructor calls that build the output instructions.

# Example Sketch


```
    ;; --- "FFI" mapping of input instructions and lowering API ---
    (type Inst extern (enum
      (Add  (a Input) (b Input))
      (Load (addr Input))
      (Const (val u64))
      ...)
      
    (type Reg primitive)        ;; virtual register number in output
    (type Insn primitive)       ;; instruction ID
    (type OwnedInsn primitive)  ;; instruction ID; type indicates we are the only user
    (type Value primitive)      ;; SSA value number in input
    (type usize primitive)
      
    ;; Extractor/constructor to go between an instruction reference and its produced value.
    ;; We can use `InsnValue` as an extractor to go from `Value` arguments to the producing
    ;; instructions, walking "up the operand tree" as needed to match trees of instructions.
    (decl InsnValue (Insn) Value)
    (extractor InsnValue ...)
    (constructor InsnValue ...)
    
    ;; Constructor to indicate that a value should be lowered into a register.
    (decl ValueReg (Value) Reg)
    (constructor ValueReg ...)

    ;; Extractor to get instruction that produces a value consumed *only* by
    ;; the currently-lowering instruction (and nowhere else).
    (decl OwnedValue (OwnedInsn) Value)
    (extractor OwnedValue ...)
    
    ;; Extractor that takes an instruction ID and provides its opcode.
    (decl Opcode (Opcode) Insn)
    (extractor Opcode ...)
    
    ;; Convenience extractors/matchers for each defined instruction.
    ;; These could be generated automatically in a build.rs step
    ;; and then included in a prelude.
    (decl Iadd (Value Value) Insn)
    (rule (and (Opcode Iadd) (InsnInput 0 a) (InsnInput 1 b))
          (Iadd a b))
    (decl Iconst (u64) Insn)
    (rule (and (Opcode Iconst) (InsnImmediate 0 imm))
          (Iconst imm))
    ; ...

    ;; --- x86 backend ---
      
    ; Note that while existing VCode instructions for x86 have explicit destination
    ; registers, and largely have two-address form with "modify" semantics (mirroring
    ; the actual machine instructions on x86), the terms produced by the x86 lowering
    ; rules have two explicit inputs and an implicit output. The constructors can
    ; insert the necessary move in a mechanical way so we can deal with purely
    ; expression-tree-like representations here.
    (type X86Inst
      (Move (a Reg))
      (Add (a Reg) (b Reg))
      (AddMem (a Reg) (b X86AMode))
      (AddConst (a Reg) (b u32))
    (type X86AMode
      (...))
      
    ;; ---

    (decl LowerAMode (Input) X86AMode)

    ; Addressing mode: const + other
    (rule (LowerAMode (InsnValue (Iadd (InsnValue (Iconst c)) other)))
      (BasePlusOffset (ValueReg other) c))
     
    ; Addressing mode: fallback
    (rule (LowerAMode input)
      (BaseReg (ValueReg input)))

    ; Main lowering entry point: this term reduces an `Inst` as an arg to an `X86Inst`.
    (decl (Lower Inst) X86Inst)

    ; Add with constant on one input. `I32Value` extractor matches a `u64`
    ; immediate that fits in an `i32` (signed 32-bit immediate field).
    (rule (Lower (Iadd (InsnValue (I32Value (Iconst c))) b))
      (AddConst (ValueReg b) c))

    ; Add with load on one input.
    ; Note the `FromInstOnlyUse` which ensures we match only if
    ; this is the *sole* use.
    (rule (Lower (Iadd (OwnedValue (Iload addr)) b))
      (AddMem (ValueReg b) (LowerAMode addr)))
      
    ; Fallback for Add.
    (rule (Lower (Iadd a b))
      (Add (ValueReg a) (ValueReg b)))  ;; lookup of constructor name is type-dependent --
                                        ;; here we get X86Inst::Add.
```
