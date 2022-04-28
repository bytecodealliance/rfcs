# Extended Patterns in ISLE

This RFC proposes an extension to the current [left-hand side
patterns](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md#left-hand-sides-patterns)
in the [ISLE
DSL](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md). The
ideas contained here are intended to:

- Address expressivity difficulties with patterns using custom
  extractors, allowing something better than [awkward
  hacks](https://github.com/bytecodealliance/wasmtime/pull/3993#discussion_r842152604);
- Remove the need to use [extractors that only incidentally match a
  particular
  type](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/x64/inst.isle#L1131-L1152)
  while really acting as predicates for something else;
- Address a longstanding feeling that [argument
  polarity](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md#advanced-extractors-arg-polarity)
  is papering over a more fundamental expressivity hole in the language
  
while acting purely as an extension, meaning that the updated language
will be:

- backward-compatible (all ISLE remains valid); and
- fully compatible with both "targets" of ISLE: generated Rust code
  and the lowering to SMT for verification.
  
If the extensions proposed in this RFC are adopted, it should be
possible to express combinations of conditions in a more natural "list
of predicates" form, without giving up the tree-of-matchers idioms
that are so natural for matching the overall operand/value structure.

## Motivation: Awkward Extractor Usage

The ISLE DSL requires the programmer to express rule-matching
conditions using a
[pattern](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md#left-hand-sides-patterns)
that contains a tree of *extractors* (along with other match
operators).

An extractor is like a reverse function call[^1]. While an ordinary
function call takes N parts and produces one result (e.g., build an
instruction with N operands), an extractor takes one value and takes
it apart into N parts (e.g., match on an instruction, yielding N
operands, each of which can match a subpattern).

[^1]: The inspiration was the Scala language's `unapply`
      functionality, which works basically the same way by allowing
      the user to add custom `match` forms.

This works well enough when we actually want to match and take apart
pieces of the input program; for example, in the left-hand side
(pattern) of the lowering rule for a hypothetical multiply-add
instruction

```lisp
(rule (lower (iadd a (imul b c)))
      (isa_madd a b c))
```

the `iadd` and `imul` extractors actually do "extract" something: they
take an `Inst` and produce two `Values`, for the two operands.

Unfortunately, there is another general kind of pattern-match that we
perform, where we want to *tell the matcher more information*
somehow. For example, we might want to know whether a shift amount in
one part of the input matches a type width in another:

```lisp
(rule (lower (has_type ty
               (load (iadd base_addr
                           (ishl index
                                 ;; `shift_for_ty` needs an *in-arg* for `ty`
                                 ;; and matches if the constant is the log2
                                 ;; of the byte-width for `ty`.
                                 (shift_for_ty <ty))))))
      (isa_load_indexed base_addr index))
```

In this case, the "input tree" that implicitly exists by virtue of the
extractors is *programmable*, or *parameterizable*, on other values
that we have already matched. We allow this to happen in ISLE with the
use of *in-arguments* (*in-args* for short), which are introduced in a
pattern with a `<` prefix followed by an expression that can use
already-matched variable names (e.g., `<ty` above). In general, an
extractor declaration can declare that some arguments are *inputs* and
some are *outputs*, which we call the "argument polarity".

Here are some examples of uses of arg-polarity in our existing
lowering rules:
- [shift
  amounts](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/aarch64/lower.isle#L58-L60)
  in aarch64, because the wrapping of the shift-amount depends on the
  type;
- ["match if sum fits in
  u32"](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/x64/inst.isle#L787-L791)
  functionality in x64, where we bind one summand and match the other
  with an extractor, taking the first summand as input and yielding the
  sum (if it fits in a u32) as an extractor result;
- [similar match-sum-if-fits](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/s390x/inst.isle#L1117-L1120) functionality in s390x;
- [parameterization of an equal-to-constant
  extractor](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/s390x/inst.isle#L943-L944)
  in s390x, where an in-arg is used to avoid defining a separate
  extractor for each different integer constant;
- [a same-register
  matcher](https://github.com/bytecodealliance/wasmtime/blob/e4b7c8a7376c0b4196262b176373b606c7b4c760/cranelift/codegen/src/isa/s390x/inst.isle#L1777-L1781)
  in s390x, which takes an already-bound register-typed variable and
  matches another if it is equal (with some behavior involving
  allocating temporary registers as well).

The DSL has the expressivity, thanks to in-args, to handle these
cases; but they are far from simple to understand. Our experience over
the past six months of actually using the DSL for everyday work is
that it takes a conscious effort to "invert" one's thinking -- even
for those of us who came up with this design! If we confuse ourselves,
then the situation for a newcomer who wants to contribute a bit to our
lowering patterns is going to be far worse.

Our proposal, then, is going to be based on this realization: there
are two kinds of extractors.  The "structural" extractors actually do
take apart e.g. an operator/instruction, and make sense in the
one-input/N-outputs paradigm. In contrast, the "predicate" extractors
check additional conditions and maybe perform some further, fallible,
computation, or else just match as a signal that a predicate is true,
with no outputs from the match. We have shoehorned the latter into the
paradigm intended for the former, but perhaps we should do something
different.
  
## Minimal Addition to ISLE: Pattern-List LHS

How do we extend the language while retaining backward compatibility?
Let us carefully consider what properties the language has now, which
we wish to keep, and which we may discard without causing too many
problems:

- ISLE currently has a strong distinction between *fallible* left-hand
  side extractors and *infallible* right-hand side constructors. This
  distinction comes with obligations and with guarantees as a
  result. The creator of an extractor is obliged to ensure that no
  side-effects result (a match is only a "query"). The ISLE programmer
  is obliged to place all of the conditions of a match in the
  left-hand side, as extractors. In return, the ISLE compiler (and
  hence really the user) does not have to worry about arbitrary-depth
  backtracking: once the left-hand side matches, we are
  committed.[^2][^3] This is a desirable complexity-limiting
  constraint.
  
[^2]: It is interesting to note that the use of side-effects to emit
      instructions for the output is actually something that underwent
      several changes during the design of the DSL, as well. The very
      early ideas for ISLE were essentially a Prolog-like system where
      side effects were explicitly used. We later determined that a
      "value-oriented" system, where the term rewriting at least
      conceptually rewrites values to equivalent values, was easier to
      manage; but in order not to refactor the whole VCode design at
      the same time, the returned value does not contain a tree or DAG
      of instructions, but rather just references to them (via
      `Reg`s), so side-effects eventually made their appearance anyway
      as we emit them via the side-effecting `emit` constructor. This
      is a property of the Cranelift glue and not necessarily
      fundamental to the DSL itself, but seems to be a reasonable
      design point that we should optimize for regardless.

[^3]: RHS is actually fallible because the ISLE execution as a whole
      is allowed to be "partial", so that we can gradually migrate to
      the language and fall back to legacy code. This is why
      constructors return `Option<T>`; the `Option` is for a different
      reason than the fallible extractors' `Option` that carries
      semantic meaning ("did I match or not").
      
- The lack of backtracking also means that we do not need any sort of
  "undo" of constructors, and constructors are free to have incidental
  side-effects (such as emitting instructions so that we can return
  their result registers, or allocating temporaries) even as the
  top-level form of the rules is one of rewriting equivalent values.
  
The above properties strongly argue that we should retain a LHS-to-RHS
rewriting-rule structure, rather than (for example) lowering ISLE to a
more generic Prolog-like form where extractors and constructors are
unified.

Given that constraint, our key proposal is laid out in two parts:
extended LHS patterns with `if-let` clauses, and a special notion of
pure expressions.

### Left-hand-side `if-let` clauses

First, we propose to extend the `(rule ...)` form to take a list of
LHS clauses. The current form, `(rule LHS RHS)`, applies the pattern
`LHS` to the value to be rewritten. We extend this with:
  
```lisp

(rule LHS_PATTERN
  (if-let PAT2 EXPR2)
  (if-let PAT3 EXPR3)
  ...
  RHS)
```
  
This means: match the input value with `LHS_PATTERN`. Then evaluate
`EXPR2` and match its result with `PAT2`. Likewise down the list,
until we have matched all `if-let` clauses. Like the rule's main
right-hand side, the right-hand side of each `if-let` is an expression
that can invoke constructors. However, these constructors (i) are
fallible, and (ii) are pure (have no side-effects). If any expression
on the right-hand side of an `if-let` fails, the rule fails and we try
to match with another rule. Likewise if any pattern on the left-hand
side of an `if-let` fails, the rule fails as well. Because there are
no side-effects from any `if-let`s, the match can fail partway through
the list without corrupting state.

This is backwards-compatible, in that rules with no `if-let` clauses
behave the same as before. In other words, the language is a superset
and has the same semantics for existing programs. It has the same
power as in-arguments, but in a more clearly expressed way. And it can
be used in a way that users may know and expect from other
pattern-matching languages, such as conditional guards, or from Rust,
with code structured as nested `match`es on values that are
extracted. And it has a structure similar to a let-chain, which is a
common idiom on right-hand sides: it lays out the match into a
sequence of steps with bindings naming the parts as we go.

### Pure and Fallible Expressions

Second, we propose to add a notion of pureness and fallibility to
right-hand side expressions and constructors so that we can allow
these expressions to be evaluated while we are still in the match
phase, progressing through the list of `if-let` clauses.

This is an extension to the language because previously, programmable
match failures had to be expressed in terms of extractors, which cause
awkward issues as described above. In other words, this proposal
brings a symmetry to the language by allowing either N-to-1
constructors or 1-to-N extractors in the match phase, as long as they
have no side-effects.
  
Note that evaluating expressions as part of the left-hand side pattern
was technically possible already, as the `<EXPR` syntax for an
extractor in-argument embeds an arbitrary evaluation. We never ran
into issues because we only ever used in-arguments with constant
integers or with already-bound variables. This language extension
would give us a safer way of reasoning about the left-hand-side
evaluations by drawing a formal line and requiring an explicit opt-in
("this constructor is safe").

This would be expressed as:

```lisp

(decl u32_fallible_add (u32 u32) u32)
;; `u32_fallible_add` can now be used in patterns in `if-let` clauses
(extern constructor pure u32_fallible_add u32_fallible_add)
```

External constructors can be marked `pure` (and are not pure by
default). Constants and enum constructors are always pure.

When a constructor is marked as pure, its return type becomes
`Option<T>`, to allow the evaluation to fail. This causes us to move
on to the next rule, as described above. Note that while internal
constructors also return `Option<T>`, in that case the optionality
corresponds to the partial mapping at the top level (i.e., ISLE can
fail to match at all, allowing fallback to legacy code).

Thus, `u32_fallible_add` above returns an `Option<u32>`, and if it
returns a `None` (presumably if the sum would not fit in a `u32`),
then this rule does not match and we move on to any others.

### Examples

* A rule that combines multiple constant `u32` offsets to a load
  instruction, but only if the sum of all of them fits into a `u32`:

  ```lisp

  (decl u32_fallible_add (u32 u32) u32)
  ;; `u32_fallible_add` can now be used in patterns in `if-let` clauses
  (extern constructor pure u32_fallible_add u32_fallible_add)

  (rule (lower (load (iadd addr
                           (iadd (uextend (iconst k1))
                                 (uextend (iconst k2))))))
        (if-let k (u32_fallible_add k1 k2))
        (isa_load (amode_reg_offset addr k)))
  ```
  
* A rule where a `if-let` clause is used to factor out a subpattern,
  perhaps for clarity:
  
  ```lisp
  (rule (lower (iadd x (load addr)))
        (if-let (addr_amode amode) addr)
        (x64_add x (RegMemImm.Mem amode)))
  ```
  
* A rule where a `if-let` clause is used to evaluate a fallible
  constructor that acts like a predicate, returning an empty value
  (matched with the wildcard `_`):
  
  ```lisp
  (rule (lower (magic_simd_op x y))
        (if-let _ (magic_simd_extension_enabled))
        (x64_magic_simd_op x y))
  ```

Note that all three of these examples could be expressed with inlined
extractors in the single main pattern, and appropriate use of in-args
to wire up the needed dataflow. But the top-level list of matches
makes it easier to naturally express patterns in logical parts.

## Alternatives Considered

### Prolog

The main alternative I considered was to fully generalize ISLE into a
Prolog-like language where left-hand and right-hand sides are the same
(merged into a list of steps or invocations instead), `match` is a
first-class operator, and extractors are just syntax sugar for a
special invocation of a function with one input and N outputs.  (In
other words, pretty much just Prolog, but with fixed-polarity
arguments rather than full unification.)

This was worth giving serious thought, given the confluence of
expressivity issues that kept occurring, and the generality would
allow for significantly clearer forms of some rules. And it would, I
think, actually not complicate the verification target too much (at
least in principle, notwithstanding existing infrastructure built up
around the rule/LHS/RHS paradigm): the basic "emit SMT clauses for
each part of the pattern / expression" pass can work over a list of
Prolog term invocations just as well as a one-LHS, one-RHS form.

I think this is technically possible, and may someday serve as a
language evolution step; but the three major reasons this is not
viable right now, and for a while yet, are:

- It implies more general backtracking. We don't yet have enough
  experience to reason through how we would even implement this, let
  alone reason about or manage the runtime complexity as we author
  cascading chains of rewrite rules. It's possible that we will want
  such flexibility in the future, at least at higher optimization
  levels (maybe for cases when we can't statically resolve the best
  path, a la superoptimization). But it's almost certainly better not
  to cross this line unless we really need to.

- It's a significant refactor. The ISLE DSL compiler, and the
  in-progress verification tool as well, are built around the current
  term-rewriting paradigm. Understanding a conceptual leap of "A
  conceptually lowers to B, which is more general" is one thing;
  twisting the whole codebase into that new direction is another.

- It's a significant cognitive shift. We have spent time internalizing
  ISLE's abstractions and best practices, and we are generally
  becoming more proficient with the DSL, producing some quite elegant
  lowering pattern rules by now (in my possibly biased opinion!). It
  is one thing for us to extend the language in a way that no one has
  to use if they don't want to; it is another to say "we now must all
  rebuild our understanding".
  
In other words, briefly, it would be too much change. ISLE generally
is working well, except for the cases where its expressive power could
be improved; so, this proposal chooses to improve its expressive power
just in those corners, in an incremental way.

### Some sort of "inverted syntax" for extractors

One concrete way that the current hackish use of extractors is awkward
is that the programmer has to "invert their brain" to see an extractor
as receiving its "return value" and returning its "arguments" (the
1-to-N pattern inherent to a match/destructuring). Perhaps some syntax
could allow for invoking an extractor in a different way, listed after
the main pattern, with inputs and outputs more clearly marked.

This idea was actually one of the kernels that led to the main
proposal above. The additional delta on top of it that makes it much
nicer is the continued use of patterns (rather than some alternative
invocation of extractors), and allowing *constructors* to be fallible
instead, if pure. This feels more honest: we really do want to do an
N-to-1 computation (which is a constructor), we just want to possibly
fail the match if it doesn't "fit" (e.g., the `u32_fallible_add`
scenario above).

### Conditional predicates on rules

The general idea here would be to introduce an analogue to Rust's
`if`-clauses on `match` arms. This takes care of the second example
above, where we use an ISA-extension predicate that currently has to
be an "extractor matching an irrelevant value"; but doesn't do
anything for other patterns that use in-arguments, so is an incomplete
solution.

## Implementation

This RFC will not be too prescriptive as to the implementation
strategy, but the general approach will likely be:

1. We add the notion of "pure" external constructors, and the ability
   to determine whether an expression is pure.
   
2. We add `if-let` clauses to the `ast` data structures, and extend
   the parser to parse them. This should almost entirely be able to
   reuse parsers for patterns and expressions.
   
3. We add `if-let` causes to the `sema` data structures, and extend
   the semantic analysis and typechecking to translate them. This
   should mostly be able to reuse the translation code for patterns
   and expressions. Typechecking is somewhat less constrained than for
   a toplevel rule, because we don't have a `decl` to start with; but
   we can typecheck the expression of each `if-let` first, requiring
   the toplevel to be a form whose type we know, and then typecheck
   the pattern given that known type context.
   
4. We lower `if-let` to IR in the `lower` pass, similarly to how we
   lower patterns now. Because `PatternInst` already has an `Expr` arm
   (to handle in-arguments), we can lower all of the `if-let`
   expression and pattern into the LHS of the IR. This is as we should
   expect, because the `if-let`s are fallible and thus part of the
   match phase.
   
5. That's it! From IR onward, everything should work without changes:
   the IR is already a linear sequence of match operations, and so
   appending some `if-let` bodies to that sequence should be perfectly
   natural.

6. At some later point, if we have moved all uses of argument polarity
   to `if-let` clauses with pure constructors instead, we can remove
   argument polarity, significantly simplifying some parts of the ISLE
   compiler. But we need not do this yet, or with any urgency;
   remaining backward-compatible now is no problem and a gradual
   transition is best.

## Open Questions

1. Most importantly, is this the right balance of expressivity (enough
   to solve our problems,) with minimality and simplicity (extending
   existing ideas as far as possible, remaining "conceptually small")?

2. Compatibility with all of our anticipated uses of ISLE -- this will
   almost certainly work with `islec`, but we should ensure that it is
   not unduly hard to update the `isle-veri` backend, and that we
   aren't precluding any other uses or analyzability later by adding
   this expressivity.
