# Cranelift: Using E-Graphs for Verified, Cooperating Middle-End Optimizations

## Summary

This RFC proposes to add a [middle-end optimization
framework](https://en.wikipedia.org/wiki/Compiler#Middle_end) to
Cranelift based on
[e-graphs](https://en.wikipedia.org/wiki/E-graph). A middle-end
optimization framework works to improve the target-independent [IR
(intermediate
representation)](https://en.wikipedia.org/wiki/Intermediate_representation),
[CLIF](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/docs/ir.md),
before Cranelift translates it to machine instructions. Cranelift
already has some basic optimizations in this category: [Global Value
Numbering (GVN)](https://en.wikipedia.org/wiki/Value_numbering),
[Loop-Independent Code Motion
(LICM)](https://en.wikipedia.org/wiki/Loop-invariant_code_motion),
[Constant folding](https://en.wikipedia.org/wiki/Constant_folding),
and most recently, a very basic form of [alias
analysis](https://en.wikipedia.org/wiki/Alias_analysis) and
redundant-load elimination. However, the current situation falls short
in several ways.

The RFC will first describe how the current situation suffers from
three problems: how to order and interleave these optimizations as we
build more of them (the *phase-ordering problem*), how to ensure they
are correct when they are all delicate hand-written code (the
*verification problem*), and how to make it easier to build up a large
body of known simplifications without tedious handwritten code. It
will then describe how an e-graph-based framework could address all
three of these problems. It will describe some novel contributions in
how to encode control flow in e-graphs (necessary in order to use
e-graphs to optimize whole function bodies) developed during initial
experimentation. Then finally it will discuss how we can use our
existing rewrite-rule DSL,
[ISLE](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md),
to describe rewrites on the e-graph.

## Motivation

A primary goal of Cranelift in 2022, as per [the
roadmap](https://github.com/bytecodealliance/rfcs/blob/main/accepted/cranelift-roadmap-2022.md),
is to improve the performance of generated code. Work on this goal so
far has mainly focused on better instruction selection in the
backends, and on transitioning to a new register allocator. We
continue to work on instruction selection, but it is increasingly
clear that mid-end optimizations are a weak spot in Cranelift as
compared to peer compilers.

We implement simple forms of many of the classical compiler
optimizations described in Allen and Cocke's [A catalogue of
optimizing
transformations](https://www.clear.rice.edu/comp512/Lectures/Papers/1971-allen-catalog.pdf)
(from 1971 -- the basics haven't changed in a while!). This includes a
GVN pass that merges redundant computations of the same value into a
single shared computation; a "simple preopt" set of rewrites that do
some constant folding (`x := 1; x + 2` becomes `3`) and algebraic
simplifications (`x + 0` becomes `x`); a loop-invariant code motion
pass that hoists computations that don't change per loop iteration out
of a loop; and some miscellaneous others. We also recently added an
alias analysis that enables us to merge redundant loads into one
instance of the load (when we can prove they always load the same
value) and rewrite some loads into a stored value we know they must
access. However, as we look to grow this collection of optimizations,
three main problems are becoming clearer.

### Problem #1: Phase Ordering

The first problem we see is one of *phase ordering*. This is a
classical compiler problem that arises to some degree in every
compiler that can rewrite code in different ways. Stated simply, the
problem is: which rewrites/optimizations do we do, in which order, to
reach the best code?

It is often not sufficient to apply each optimization pass only
once. For a simple example, consider the following CLIF fragment:

```plain
    v1 := ...
    v2 := ...
    
    v3 := ishl v2, 4
    v4 := iadd v1, v3
    v5 := load v4+8
    v6 := iadd v5, v2
    v7 := load v6

    ;; ... some series of operations with no memory side-effects
    
    v100 := ishl v2, 4
    v101 := iadd v1, v100
    v102 := load v101+8
    v103 := iadd v102, v2
    v104 := load v103
```

We would like to be able to use the combined reasoning power of GVN
and alias analysis to prove that `v104` is the same value as `v7`, and
rewrite all uses of it to `v7` so that we can delete `v100` through
`v104` entirely. However the chain of reasoning to do this requires
something like: (i) GVN to prove that `v100` is an alias of `v3`, and
so `v101` is an alias of `v4`; (ii) then given that, alias analysis
and redundant load elimination to prove that `v102` is an alias of
`v5`; (iii) then given that, GVN again to prove that `v103` is an
alias of `v6`; (iv) then given that, alias analysis and redundant load
elimination again to prove that `v104` is an alias of `v7`.

In other words, this chain of reasoning requires cooperation between
GVN and redundant load elimination with two "roundtrips". If these
were entirely separate passes that scan over the entire function -- as
they are today in Cranelift -- we would have to invoke GVN over the
*whole function* once, then redundant load elimination (RLE), then GVN
again, then RLE again. We do twice as much work as one might have
originally expected to do to apply "all optimizations".

What is worse, this requirement for redundant re-application of a pass
is bounded in the worst case only by the length of the code (if one
wants to capture all opportunity).

When we had only GVN, LICM, and simple forms of constant folding, this
was somewhat manageable: the [`Context::compile()`
method](https://github.com/bytecodealliance/wasmtime/blob/c7be93753ac9fcdff816eaccad00986a6ac06b27/cranelift/codegen/src/context.rs#L131)
invokes pre-opt, LICM, GVN, dead-code elimination (DCE), and
constant-phi elimination in that order, each pass once. With the
addition of alias analysis, we added an invocation of RLE followed by
one more round of GVN right at the end. But it was far from clear that
this would be enough; and one can easily construct examples like the
above where it is not. More aggressively optimizing compilers have
elaborately-developed and highly-tuned default sequences of passes:
e.g. LLVM's
[PassManagerBuilder.cpp](https://github.com/llvm-mirror/llvm/blob/2c4ca6832fa6b306ee6a7010bfb80a3f2596f824/lib/Transforms/IPO/PassManagerBuilder.cpp)
is 1136 lines of (carefully-commented, at least) code to build a list
of passes; gcc's
[passes.def](https://github.com/gcc-mirror/gcc/blob/cf79b1117bd177d3d4c6ed24b6fa243c3628ac2d/gcc/passes.def)
is likewise 538 dense lines sequencing several hundred unique
passes. GCC appears to invoke CSE (common-subexpression elimination)
at least three times plus two special variants: `cse`,
`cse_after_global_opts`, `cse2`, `cse_sincos`, `cse_reciprocals`. We
should not aspire to replicate this approach!

We ideally need some way of *interleaving passes on a fine
granularity*: we should be able to, for a single expression, see that
we can apply a series of rewrite rules as above without running the
whole function through the passes multiple times. This requires some
central dispatch facility that understands the passes (or "rewrite
rules") in a unified way.

### Problem #2: Verifiability

The second problem with the current approach is that in general,
handwritten passes are not trivial to *verify*. We made verification a
first-class goal of our instruction-selection framework design, and as
a result, we have an ongoing effort to do exactly that (with formal
methods using an SMT solver). Likewise, if we want to verify that our
optimization passes preserve the meaning of the user's code, we should
take the same approach here and consider verification a first-class
focus.

It is generally well-studied how handwritten compiler code can
introduce subtle bugs: for example, the
[Alive](https://www.cs.utah.edu/~regehr/papers/pldi15.pdf)
verification engine for LLVM was explicitly built to find bugs in the
InstCombine subsystem of LLVM, which does rewrites of the IR with [a
significant body of hand-written
C++](https://github.com/llvm-mirror/llvm/tree/master/lib/Transforms/InstCombine). This
work found 8 bugs in 334 hand-coded transforms (2.4%), a serious issue
when any given codegen bug could be catastrophic (e.g. cause a CVE).

Ideally, we would have a way to express most or all code transforms in
a declarative way, where it is (i) easier to ensure when writing, and
see by inspection later, that the equivalence holds; and more
importantly (ii) eventually use an automated or semi-automated
workflow to prove the rewrite(s) correct against a semantics for our
IR, CLIF. Such a verification task is significantly easier when we
process only the equivalence and not the incidental details of an
associated hand-written transform.

### Problem #3: Ease of Contributing Optimizations

The third problem with the current approach is that it is not easy to
write new optimizations. When every optimization starts "from
scratch", with a lot of boilerplate to iterate over blocks and
instructions, build up knowledge, and then mutate values in place, the
"activation energy" to add a simple rewrite expressed declaratively in
one line -- say, a strength-reduction rule like `8*x => x << 3` -- is
often too high to bother.

Ideally, we should have a way of using the same declarative principles
and tools we have developed for instruction lowering with
[ISLE](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/isle/docs/language-reference.md)
in order to write a pattern just like the above, or at least close to
it. For example, to sketch a possible ISLE encoding of the above, one
might write:

```lisp
(rule (simplify (imul x (power_of_two log2y)))
      (ishl x log2y))
```

Moreover, if rules could be written this way, they would be easier to
produce not only for humans but for *optimization-discovery tools* as
well. For example, the
[Peepmatic](https://github.com/bytecodealliance/wasmtime/pull/1647)
tooling included tools to "harvest" potential patterns to rewrite from
real programs, and then use the superoptimization engine
[Souper](https://github.com/google/souper) to find rewrite rules
offline. We could revisit such tooling and use it to build a
significantly more effective body of rules semi-automatically.

## Proposal: E-graph-based Optimization

This RFC proposes the adoption of a new middle-end optimization
framework built around
[e-graphs](https://en.wikipedia.org/wiki/E-graph), and rewrite rules
on those e-graphs. We will summarize what e-graphs are and how they
work, then work through a series of adaptations needed to allow a
Cranelift function body with control flow to be represented in an
e-graph. In the subsequent section we'll see how this data structure
will actually be used for optimization rewrites.

### Recap: E-graphs

An e-graph, or "equivalence graph", is a representation of a set of
value nodes, each of which may use other nodes as arguments, that
allows *equivalences* between these nodes to be recorded. It
efficiently represents *all possible expressions* for a value, given
these equivalences. It has normalizing properties: "adding" the same
expression to an e-graph multiple times will result in exactly the
same ID.

An e-graph consists of a set of *classes* (or "eclasses"), each of
which has a set of equivalent *nodes* (or "enodes"). An eclass has an
identity (in our implementation based on the `egg` library, simply a
numeric index). Enodes, however, do not have an identity or name. A
"value" in the program is represented by an eclass ID.

Aside from a map of eclass ID to eclass data (with list of nodes), an
e-graph contains (i) a hashmap of node contents to eclass ID, to allow
for deduplication ("hash-consing") when adding new nodes; and (ii) a
[union-find](https://en.wikipedia.org/wiki/Disjoint-set_data_structure)
data structure to enable efficient merging of eclasses and looking up
canonical eclass IDs post-merge.

The algorithmic and implementation details of e-graphs are beyond the
scope of this RFC, but the [egg
paper](https://dl.acm.org/doi/10.1145/3434304)[^1] gives a good
overview, including some novel contributions of that paper to make
e-graph updates asymptotically more efficient.

[^1]: M Willsey, C Nandi, Y R Wang, O Flatt, Z Tatlock, P
      Panchekha. "egg: Fast and Flexible Equality Saturation." In
      POPL 2021. https://dl.acm.org/doi/10.1145/3434304
      
The key relevant facts here are:

- An e-graph lets us build expressions out of *nodes*, each of which
  contains references to other nodes. This is very much like a
  sea-of-nodes IR.
  
- An e-graph lets us equate one node with another, such that when we
  examine the value (eclass ID), we can see both nodes and eventually
  choose either for code generation.
  
The first fact is relevant to how we express a function body in an
e-graph, and we will explore that question in this section. The second
fact is relevant to how we use an e-graph to progressively rewrite the
function, and we will explore that in the next section.

### Control Flow in E-graphs

Traditionally, e-graphs have been used to represent expression nodes
in a control-flow-free representation (or at least, we have not been
able to find an example of a system that embeds control flow into an
e-graph). Cranelift function bodies expressed in CLIF have basic
blocks of instructions in a control-flow graph (CFG); we thus need
some way to embed the information in the CFG within the e-graph, so
that when we generate code from the optimized function body within the
e-graph, we retain equivalent control flow.

There have been numerous *graph-based IRs* of other forms that *do*
represent control flow, however. Two examples are the Program
Dependence Graph (PDG) and Value-State Dependence Graph (VSDG). Both
of these essentially represent control dependencies uniformly as they
do data dependencies: blocks or regions have a "start node" or
equivalent, and operators depend on the node as well as their data
arguments.

However, putting some sort of control-flow block or region node into
the e-graph is only the start: we need (i) to ensure that
optimizations are able to observe and rewrite expressions that span as
much control flow as possible, and (ii) to have a way to re-extract
the control flow and express computation in a linearized order at the
end of the process.

These two goals seem to be at odds with each other. First, to be as
strongly-normalizing as possible, the e-graph would ideally *not*
incorporate control-flow information into "pure" (side-effect-free)
enodes, and ideally would not have any other form of separation
between the operators in different blocks. For example, the operator
`iadd v1, v2` in some block `B1` would *ideally* be represented by the
same eclass as `iadd v1, v2` in block `B2`. That way, any
optimizations that simplify the expression apply in both
places. Furthermore, if the operands `v1` and `v2` were defined in
some earlier block, we want rewrite rules to be able to seemlessly
match across both nodes.

On the other hand, if nodes that represent instructions lose all
information about their original place in the CFG, we have a very
tricky problem when we try to generate code: we have to decide a
*schedule* for the nodes, i.e., when we compute each one.

Let's list a few properties that we want our representation and the
corresponding scheduling approach to have:

- We want to subsume GVN (global value numbering). That is, if we have
  the following CLIF:
  
  ```plain
  block0:
    v3 := iadd v1, v2
    brif block1
    jump block2
    
  block1:
    v4 := iadd v1, v2
    ...

  block2:
    ...
  ```
  
  then we would like for `v4` to be represented by the same node as
  `v3`, and if control flow jumps from `block0` to `block1`, we should
  use the value computed in `block0` without recomputing it in
  `block1`.
  
- We want to *avoid* creating partially-dead code. This means that we
  should not transform a program such that an operator that was not
  computed in the original program is computed in the optimized
  program. Consider the following example:
  
  ```plain
  block0:
    br_table v1, [block1, block2, block3]
    
  block1:
    v3 := iadd v1, v2
    ...
    
  block2:
    v4 := iadd v1, v2
    ...
    
  block3:
    ...
  ```
  
  In this program, we should *not* compute `iadd v1, v2` in `block0`,
  but only if control flow reaches `block1` or `block2`.
  
  Note that this implies that a single node may be computed at
  *multiple* locations in the final program produced from the e-graph.
  
- We want the node for `iadd v1, v2` to encode *only* that
  information, and not its original location or locations, because
  otherwise we break the e-graph hash-consing abstraction (where a
  node's identity is exactly its contents).
  
We will see below that with an algorithm we call "scoped elaboration",
it is possible to efficiently compute a schedule for nodes that
satisfies all of these properties. The representation that this
requires in the e-graph corresponds to the following e-node type:

```rust
pub enum Node {
    /// A blockparam. Effectively an input/root; does not refer to
    /// predecessors' branch arguments, because this would create
    /// cycles.
    Param {
        /// CLIF block this param comes from.
        block: Block,
        /// Index of blockparam within block.
        index: u32,
        /// Type of the value.
        ty: Type,
    },
    /// A CLIF instruction that is pure (has no side-effects). Not
    /// tied to any location; we will compute a set of locations at
    /// which to compute this node during lowering back out of the
    /// egraph.
    Pure {
        /// The instruction data, without SSA values.
        op: InstructionImms,
        /// eclass arguments to the operator.
        args: ArgVec,
        /// Type(s) of result(s).
        types: TypeVec,
    },
    /// A CLIF instruction that has side-effects or is otherwise not
    /// representable by `Pure`.
    Inst {
        /// The instruction data, without SSA values.
        op: InstructionImms,
        /// eclass arguments to the operator.
        args: ArgVec,
        /// Type(s) of result(s).
        types: TypeVec,
        /// The original instruction. We include this so that the
        /// `Inst`s are not deduplicated: every instance is a
        /// logically separate and unique side-effect.
        inst: Inst,
        /// The source location to preserve.
        srcloc: SourceLoc,
    },
    /// A projection of one result of an `Inst` or `Pure`.
    Result {
        /// `Inst` or `Pure` node.
        value: Id,
        /// Index of the result we want.
        result: usize,
        /// Type of the value.
        ty: Type,
    },
}
```

The function body is then represented by an e-graph consisting of
eclasses of these enodes, together with a *side-effect skeleton*.

The side-effect skeleton is a subset of the original CFG: it retains
the original graph of blocks, but within each block, it names only the
eclass IDs for non-pure (side-effecting) instructions, in their
original order.

Thus, in a sense, we keep the part of the CFG that will not change "on
the side" (changing this structure is out-of-scope for expression
rewrites in any case), and allow the pure part of the program to
"float" over this skeleton. We rewrite it as needed, then eventually
copy the floating nodes into one or more locations in the final CFG.

### Cycles from an Acyclic Graph

In the node definition above, some care was taken to ensure that the
original e-graph, as constructed from the CLIF and with every enode in
its own eclass, is acyclic. (Cycles here are defined in terms of the
graph obtained by taking an eclass's enodes' argument eclass IDs as
out-edges of that eclass.) This arises mainly because block parameters
("blockparams") are the only means to create cycles in the dataflow
graph; the SSA dataflow is otherwise acyclic because of the
defs-dominate-uses property. By translating block parameter values
into "terminal" enodes (enodes without any arguments), we remove any
cycles in the original dataflow.[^2]

[^2]: Note that omitting predecessor blocks' inputs to blockparams in
     the e-graph does have implications for optimizations that are
     possible. In particular, we cannot write a "remove redundant
     phis" rewrite rule (if all predecessors' inputs to a blockparam
     are one value `v`, rewrite that blockparam as an alias to
     `v`). We could however restore these edges eventually and write
     such a rule, as long as we are careful to ignore the edges (i.e.,
     effectively remove the edges, and use actual blockparams to link
     up the dataflow) once we "elaborate" back into a CFG.

This acyclic nature of the e-graph is very desirable: it allows us to
traverse the e-graph and generate values recursively[^3] with a
guarantee that this algorithm will terminate. If at all possible, we
would like to preserve this property; and in fact, the "elaboration"
procedure we describe in the next section requires it.

[^3]: As manual recursion with an explicit stack, anyway, as we want
      to avoid literal stack recursion in practice (as we do
      throughout the rest of Cranelift).

However, as soon as eclasses begin to merge, cycles may occur. To see
how, consider the following e-graph:

```plain
          A
         /  \
        B     C
      / \      \
      D  E      F
    /
   G
```

If B and G were merged we might get:

```plain
         A
        /  \
 .---B,G   C
 ^  / |      \
 | D  E       F
 \_|
```

In general, condensing two or more nodes into one node in an acyclic
graph will produce a cycle if a path existed from one node to the
other prior to the merge.

Seen another way, a rewrite rule that rewrites an entire expression to
just part of that expression (e.g., `x + 0 => x`) can be seen as
"generative" (produces infinite growth) if run in reverse. Because an
e-graph captures many equivalent expressions, rewriting "down" one
step (to a smaller expression) produces an e-graph that is
indistinguishable from rewriting "up" one step. (Start with `x + 0`
and merge that eclass with `x`; one will get the same e-graph as
starting with `x` and merging with `x + 0`.) The e-graph thus
represents infinitely many possible expressions whenever a rewrite
applies that makes an entire expression equivalent to one part of
it. ("A path exists from one node to the other prior to the merge",
specifically a path down the expression tree.) This is extremely
counter-intuitive, because one might expect a rule `x + 0 => x` to
only produce "smaller" expressions. The subtlety arises because of the
monotonically-growing nature of the e-graph.

### Lowering Step One: Cycle Removal

Understanding *why* the cycles occur is the first step to removing
them. Namely, we saw above that an acyclic e-graph acquires cycles
whenever we have a rule that rewrites a whole expression to part of
that expression.

This is great news: it means that there is still an acyclic subgraph
inside of the e-graph if we just choose one of the several possible
equivalent enodes at an eclass in the cycle.

Returning to the example from above

```plain
         A
        /  \
 .---B,G   C
 ^  / |      \
 | D  E       F
 \_|
```

we can *delete* the `B` enode within the `B,G` eclass in order to obtain

```plain
          A
        /  \
      G     C
             \
  E           F
```

where the now-unreferenced eclass containing `E` will subsequently be
ignored.

How do we do this in the general case? We define an algorithm based on
a *DFS traversal* of the e-graph that runs after all rewrites have
been done in order to recover the acyclic invariant, allowing for
codegen.[^4]

[^4]: Actually, it serves as the extraction procedure for the e-graph
      at the same time: that is, it not only deletes enodes to remove
      cycles, but it actually chooses just one enode per eclass so we
      have just one canonical version of each expression. But this is
      not necessary for the essential cycle-removal property and we
      may in fact want to remove the explicit extraction in the future
      if instruction selection becomes more flexible.
      
The rough algorithm is as follows: we track state for each eclass as
one of `None`, `Visiting`, `Visited { cost, chosen_node_idx }`,
`Deleted`. We do a DFS from each "root" (defined by the side-effecting
skeleton). When we visit a node, then:

- if `None`, transition to `Visiting`. Visit each argument of each
  enode in turn. The visit returns the extracted cost of that
  argument, or "deleted", in which case the enode with that argument
  must be deleted. We compute the minimum cost based on sum of all
  argument costs and a fixed operator cost (a "greedy extractor" in
  e-graph terms). If all enodes in the eclass are deleted, we mark the
  eclass `Deleted` and return this state; otherwise, we update the
  node state to `Visited` and reflect the cost and the enode with that
  lowest cost.
  
- if `Visited`, we already know that everything reachable from this
  eclass is acyclic and we have its cost, so we return it.
  
- if `Deleted`, we already know that this eclass has been deleted, so
  we return that status; in turn this will result in the removal of
  the enode that used it.
  
- if `Visiting`, we have found a cycle! As with `Deleted`, we return
  "deleted". However, note that we do *not* delete the eclass we
  reached; it may have other viable enodes that do not reach a cycle.
  
This algorithm has been described recursively, but in the final
implementation we plan to write it with an explicit stack rather than
implicitly relying on the callstack, so that user-controlled input
cannot cause a stack overflow.

### Global Code Motion and Node Placement

The next problem to solve, and in fact the main one aside from actual
rewrites (optimizations), is to bridge the semantic gap from the
e-graph and a traditional CFG-based IR in order to allow the compiler
backend to produce linearized code.

At a high level, there is a design choice to make: do we "pin down"
operators to a particular location in the CFG (partially ordered
within a block, or even in a specific sequence order), or do we
represent only the operator and then somehow work out when to compute
it later?

Attaching a location to each enode is quite tempting. There are
several ways we could do this:

1. We could build an e-graph per basic block, and separately optimize
   each basic block. Then once we are done, we know which eclasses we
   need (the original side-effecting instructions in the original
   order), and we have a partial order based on dataflow dependencies,
   so we can do a topological sort and produce a linear sequence of
   instructions.
   
2. We could explicitly wire up control flow "predicates" in the graph,
   so even a pure operator depends on some node (that represents
   "control has reached a point where this operator must eventually
   run"). One might call these "predicates". The PDG (Program
   Dependence Graph) approach and [R]VSDG ([Regionalized] Value State
   Dependence Graph) approach both fall into this category, broadly
   speaking, though the details are slightly different (predicates
   vs. nested regions).[^5]
   
[^5]: See
      [Alan C. Lawrence's dissertation](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-705.html)
      for much more exploration of this space!
   
3. We could embed a block identifier in each enode, so otherwise
   identical enodes from different blocks are not deduplicated, but
   enodes from different blocks can still be observed together by
   rewrite rules. We could then write an explicit rule that does GVN:
   if an enode exists at some block B1 and another block B2, and B1
   dominates B2, then make the eclasses equivalent.
   
However, each of these approaches has downsides:

1. If we separately optimize each basic block, we lose significant
   opportunity. There is no reason otherwise that a rewrite rule
   should not be able to observe expressions that are computed across
   basic blocks, and most optimizing compilers are able to do
   e.g. constant folding across the function body, not just inside a
   single basic block.
   
2. Predicates and/or regions implied by a PDG or [R]VSDG are difficult
   to transition into and out of when the rest of the compiler works
   in terms of traditional control-flow graphs. It can be done -- for
   example, the Relooper algorithm exists to extract structured
   control flow out of an arbitrary CFG, and Bahmann et al.[^6] show
   that one can recover the original CFG out of an RVSDG
   perfectly. Regarding the interaction of e-graph merging and RVSDGs,
   a recent [prototype](https://github.com/jameysharp/optir) by
   @jameysharp shows a working example. Still, if we can help it,
   ideally we do not want to pay the cost of the semantic gap in
   control-flow representation (relooping and then recovering a
   CFG). Such complexity yields both bugs and slow compilation!

   
3. Starting with a separate copy of an enode per block, and then
   merging explicitly to do GVN, seems significantly suboptimal: the
   secondary data structures to index the enodes on
   everything-but-location, and the pass to merge them appropriately,
   seem both costly and more complex than today's GVN. There is also
   the question of where to place newly created nodes (which block to
   assign them to) when rewriting rules produce them.

[^6]: Helge Bahmann, Nico Reissmann, Magnus Jahre, Jan Christian
      Meyer. "Perfect Reconstructability of Control Flow from Demand
      Dependence Graphs." ACM Transactions on Architecture and Code
      Optimization (TACO), vol. 11, no. 4,
      article 66.
      [pdf](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43246.pdf)

Ideally we could do something better: enodes for pure operators should
not encode location at all, *and* we should find an efficient way to
generate them in the correct location(s). Fortunately, we have
discovered just such a way, and the next section describes it!

### Lowering Step Two: Scoped Elaboration

We now describe an algorithm we call "scoped elaboration" that
generates linearized code in a traditional CFG given a "side-effect
skeleton" and an e-graph with *location-free* pure nodes floating
amongst the side-effecting instruction nodes.

To understand it, let's first think about what is needed in the case
without control flow. We start with an extracted e-graph (one enode
per eclass), so from now on we can refer to just "nodes" in a DAG (in
the "sea of nodes") sense. We can get the value of any node by first
getting the value of its arguments (recursively), generating
instructions needed to do so, then generating the instruction for that
node. In other words, this is analogous to a postorder traversal on an
AST. When the e-graph is a DAG rather than tree, we can benefit from
the subexpression sharing by *memoizing*: we record the value names in
the output (e.g., SSA values or virtual register names) that
correspond to nodes we've already generated.

In pseudocode, the algorithm for a single block (no control flow) is:

```python
def get_value(node, computed_values):
  if node in computed_values:
    return computed_values[node]
    
  args = [get_value(arg_node, computed_values) for arg_node in node.args]
  
  result_value = emit_inst(node.opcode, args)
  computed_values[node] = result_value
  return result_value
  
def codegen_block(side_effect_roots, computed_values):
  for node in side_effect_roots:
    get_value(node, computed_values)

# to start with, we have an empty memoization map:
codegen_block(block_roots, {})
```

Now, the key step: to support control flow, we need to (i) steal a
data structure from GVN, the "scoped hashmap"; and (ii) remind
ourselves of SSA's key invariant, that definitions dominate uses.

We start with the domtree (dominator tree), a data structure computed
from the CFG where a block B1 is an ancestor of block B2 in the tree
if and only if B1 dominates B2.

Then we visit blocks in the domtree, and in *preorder*, perform the
above block-local scheduling for all side-effects in the block. We
then visit children and do the same.

At each level, we push a scope in the scoped hashmap. The effect of
this is that memoized eclass-to-value mappings are valid for a block
and its children in the domtree. These are exactly the blocks
dominated by that block, which per the SSA invariant, are those blocks
where that value can be used. When we return up the tree, we pop a
scope. That's it!

This algorithm is closely related to GVN, and this is not an accident:
it is designed to produce exactly the value consolidation that GVN
does. In particular, if a node's value is called for in some block B1,
and later in B2 where B1 dominates B2, the uses of that value in B2
will use the value already computed in B1.

The result of this traversal is that we generate nodes' values "as low
as possible" in the domtree, so we do not have any partially-dead code
(i.e., we do not hoist up the tree when the value is never used in
some paths from that point). At the same time, we maximally reuse
values that are already computed. We may generate a given node
multiple times, at multiple points in the tree where it is used; but
never more than necessary.

We call the memoized recursive postorder-instruction-emission visit of
the node DAG above the "elaboration" process, and so we call the
version with control flow "scoped elaboration".

In pseudocode, it looks something like:

```python
def codegen_func(domtree, side_effects_by_block):
  scoped_map = ScopedHashMap()
  elaborate_block(domtree, side_effects_by_block, domtree.root(), scoped_map)
  
def elaborate_block(domtree, side_effects_by_block, block, scoped_map):
  scoped_map.increment_level()
  
  codegen_block(side_effects_by_block[block], scoped_map)
  
  for child in domtree.children(block):
    elaborate_block(domtree, side_effects_by_block, child, scoped_map)
  
  scoped_map.decrement_level()
```

### Round-tripping Through E-graphs Subsumes GVN (and LICM?)

With the encoding scheme above, cycle elimination, and scoped
elaboration, we have a practical way of converting CLIF to an e-graph,
and then converting that e-graph back to CLIF. When in e-graph form,
the computation's "pure" operators (the ones that most expression
rewrites can replace) are completely independent of the control flow
of the function, and can be rewritten, merged, etc., at will, just as
if there were no control flow at all. Furthermore, the process of
making this roundtrip subsumes GVN: values are naturally deduplicated
by the has-consing in the e-graph, and scoped elaboration uses an
initial copy of a value in place of any other copies it dominates.

With some tweaks, this lowering back into a CFG of linearized code can
*also* subsume LICM. A general sketch: first, mark the nodes in the
domtree that correspond to the loop nest. (The loop headers will be a
subset of all block nodes, and one block is an ancestor of another in
the loop-nest tree if and only if it is an ancestor of the other in
the domtree.) When elaborating a node, record the lowest loop-nest
level in the domtree at which the args are defined. If this is higher
than the current loop nest (i.e., we are elaborating a block inside a
subloop, and we only now are seeing a use of a value that forces
codegen of a node), place it in the loop preheader of the outermost
loop below the definition level of the args, and insert it into the
scoped hashmap at that level.

(This is a long way of saying: do traditional LICM, but eagerly while
placing nodes in a forward pass, rather than in a fixpoint loop
hoisting already-placed instructions out of a loop.)

### Prototype: CLIF to E-graphs and Back Again

A prototype exists!
[wasmtime#4249](https://github.com/bytecodealliance/wasmtime/pull/4249)
is a PR for an integration with the `egg` e-graph library that
round-trips CLIF through an e-graph and back at the tail end of the
middle-end optimization pipeline, just before lowering to CLIF. It
works, to the extent that all tests pass and it can successfully
compile and execute SpiderMonkey.wasm in Wasmtime.

It is not yet optimized or polished, in several ways: it uses
callstack recursion in several places for prototyping simplicity; it
rips out and then rebuilds all CLIF instructions, rather than keeping
them in-place (and just rewriting their ordering links) for the common
case that a node is elaborated only once; and there are probably a lot
of micro-optimizations that can be done. Nevertheless, it served as a
useful means to prove out the "scoped elaboration" approach and show
that it is possible to roundtrip through an e-graph.

## Rewrites in an E-graph

Now that we have described how we propose to build the e-graph and
lower out of it, we will describe how we actually *use* it to optimize
the fucntion body.

### Optimizations as Rewrites

The key operation that an e-graph enables is a *value rewrite*. The
action of merging two e-classes when they are "equivalent" (values are
always equal) is useful mostly as a way to equate a newly-added
rewritten variant of a value to the original value.

In other words, the main flow of program optimization is:

- Find a value in the e-graph for which we know a "better"
  (cheaper-to-compute) version;
- Add that better version as e-nodes in the e-graph (creating new
  e-classes or using existing ones depending on whether each e-node
  deduplicates to an existing one or not);
- And finally, *merge* the old and new values into one e-class (uusing
  the union-find data structure).
  
This is different from a traditional approach that mutates the IR
directly in a very important way: it *preserves both versions* of the
expression. In contrast, a traditional optimization pass will often
replace all uses of the old expression with uses of the new one. Some
rewrites, such as constant folding, are probably unambiguously good to
do (always result in cheaper computations); but for others, this is
very far from the case.

Importantly, allowing for rewrite rules that may or may not make the
code better allows rewrite rules to express *possibilities* rather
than *simplifications*. These possibilities may include newly
applicable uses of other rules. For example, applying an
*associativity* rule (`x + (y + z) == (x + y) + z`) by itself will not
result in fewer computations; but it may open other
possibilities. (For example, constant folding cannot simplify `1 +
(2 + x)`, but it *can* simplify `(1 + 2) + x` to `3 + x`.)

The overall strategy is thus to *build a large body of rewrite rules*
that (i) express all possible identities that we know about, and (ii)
are "building-block" rewrites that can compose into more significant
simplifications.

The above description may evoke a design very centered on *algebraic*
rules (associativity, communitativity, identity elements (`x + 0 ==
x`), and the like), but note that many different kinds of
optimizations can be expressed as expression rewrites with some
metadata input for additional conditions. For example, we can express
store-to-load forwarding as simply rewriting a load to the appropriate
stored data, as long as we provide the "last-store" information to
rewrite rules.

### Runtime and Optimization Fuel

So far, we have described the rewrite system in a way that implies
broad exploration of many possibilities. Basically, it is a graph
search (where graph nodes are particular program variants and graph
edges express one rewrite) for the cheapest version of the program.

Since this search spans a combinatorially-large space, we will need a
way to prevent it from running for exponential time (and growing the
e-graph to take exponential memory). We will thus need a system of
"optimization fuel" that limits rewrite steps overall and possibly
per-rule or per-eclass. It is not yet certain what policies will work
best; this is a question that we will need to experiment with. We
simply wish to acknowledge now that it *will* be a problem that we
will need to solve.

### Expressing Rewrites: ISLE or egg Patterns?

In order to express e-graph rewrite rules, we will need to define or
adopt a language in which to write them. Within the Cranelift project
we already have a fairly effective pattern-matching DSL, ISLE, which
we use for our instruction selector descriptions. It thus seems fairly
clear from a complexity-management point of view that to avoid
proliferation of multiple ever-so-slightly different rewrite systems,
we should strive to reuse ISLE.

The main question seems to be: should we use the existing ISLE
compilation strategy and somehow build a toplevel rewrite driver that
invokes an ISLE "constructor", with the appropriate glue to match on
e-classes / e-nodes, or should we statically translate the ISLE rules
into a form that the `egg` e-graph library can use with its own
rewrite engine?

There are advantages to both approaches. Using `egg`'s rewrite
functionality directly would likely be a short-term win: it is known
to work, and the library is already incentivized to make this fast.

However, it is likely that we will have somewhat different
optimization constraints and needs than the average `egg` user, in the
long run. The library is optimized for ease-of-use, and it is apparent
in several places that it is not especially careful to e.g. avoid
allocations; nor does it appear to do extensive cross-rule-pattern
optimization, or allow for static optimization of rule-matching code
by generating Rust ahead of time, for example, as ISLE's metacompiler
does. If we are to make e-graph rewrites fast in an environment where
JIT-level compilation speed is necessary, we are likely to need high
control.

This RFC thus proposes that we drive rewrites by calling into
ISLE-generated Rust code, as we do for instruction lowering today. It
turns out that there is a way to extend ISLE extractor semantics very
slightly in order to allow for a natural binding of e-class, rather
than e-node, matching, which we now describe.

### Minimal ISLE Change: Multi-Extractors

The basic problem we need to solve is that while ISLE matching on CLIF
is matching a single operator per value, hence an AST-like DAG, ISLE
matching on an e-graph could match any of the e-nodes in a given
e-class. Said another way: we can speak of an e-class ID rather than
an SSA value as the fundamental atom in the IR (so `egg`'s `Id` type
rather than `cranelift-codegen`'s `Value`); but what instruction
(e-node) does `def_inst` give us as the definition of an e-class `Id`?

This RFC asks: why not all of them? Then we "just" need to somehow run
the matching logic for each possible candidate. The first matching
rule still wins.

Consider the definition of `def_inst` in the prelude today:

```lisp
(decl def_inst (Inst) Value)
(extern extractor def_inst def_inst)
```

We can adapt that in the mid-end universe to

```lisp
(decl def_inst (InstData) Id)
(extern extractor def_inst def_inst)
```

where `InstData` is the data for an e-node (containing `Id` args), and
`Id` is the e-class ID. The usual implicit conversions will be in
place as well so we get `def_inst` for free when writing tree
matchers. Then if the user writes a rule

```lisp
(rule (rewrite (iadd (iconst 0) x))
      x)
```

the generated Rust code from the ISLE may contain a snippet like:

```rust
    // Sketching simplified details here; the control flow is the
    // important part:
    //
    if let Some(inst_data) = C::def_inst(ctx, value) {
        if let InstData::Iadd { ... } = inst_data {
            // ...
        }
    }
```

However, if we define `def_inst` as

```lisp
(extern extractor multi def_inst def_inst)
```

then instead we get

```rust
    // One key difference: `while let` loop!
    //
    let mut iter = C::def_inst(ctx, value);
    while let Some(inst_data) = iter.next() {
        if let InstData::Iadd { ... } = inst_data {
            // ...
        }
    }
```

In other words, `def_inst` returns an iterator, which can yield
multiple matches (extractor results). For each we evaluate the
sub-trie; a full match (reaching a leaf node) always returns, hence
exits the loop early. This is just a multi-attempt generalization of
today's control flow.

#### Efficiency

What are the efficiency implications of this design? In general if the
rules are mostly "sparse", then we can expect a constant-factor
increase in match evaluations at any given node in the trie of
rules. By "sparse" we mean roughly that at any given e-class, we only
expect one operator, and so we won't do a deep traversal of the rule
trie for multiple e-nodes in an e-class. This is only unlikely to be
true at the root of the matching, where we expect to attempt a rewrite
on each e-node of the e-class because we'll have some rules for any
given opcode.

But it is possible for this to blow up combinatorially: in general, we
are iterating implicitly over all possible AST expansions of the
e-graph and trying to match, and the e-graph can compress an
exponentially-large set of expressions into a linear-size data
structure.

There are two general techniques we could use if this becomes an
issue. First, we could propagate "opcode" or more specific "shape"
hints *into* the `def_inst` multi-extractor. In other words, at a
given node in the rule trie, we might know that we can only match
`iadd` opcodes, or better, `iadd` with `iconst` as first child. We
could then index in each e-class by opcode (in `egg` the e-nodes are
actually sorted, which would allow this filtering efficiently). One
could even analyze the "shape" in this sense (or rather, set of
shapes) of every e-class as an e-graph analysis, and use it to
pre-select a filtered rule trie at the root.

The other general technique to apply is memoization. If we find that
it becomes more expensive to match helper(s) at any given e-class, for
example extractors that compute "is a constant" or other value
properties, we could add an ISLE feature to memoize such
"queries". This is analogous to the idea of an e-graph analysis
computing properties on e-classes rather then e-nodes, but with an
additional implicit sense of laziness.

It is likely best to move forward without these techniques at first,
and see how well compile-time fares. It is likely still to be fast at
least at first, with a small set of rewrite rules. More advanced
efficiency techniques may become necessary only as we grow our rule
body, giving us some time to study the issue and work out the best
approach.

### ISLE Bindings

The general approach will be to construct a new ISLE environment with
a slightly different prelude than for lowering (we can factor much of
the prelude into a shared common subset). This prelude will include:

- A multi-extractor `def_inst` as described above that allows for
  matching on the e-graph;
  
- Auto-generated extractors for every (pure) CLIF operator that work
  just as in the instruction-selection ISLE environment, e.g. `(iadd a
  b)`, but are defined on `Id`s rather than `Value`s;
  
- Auto-generated *constructors* for every (pure) CLIF operator that
  add the given operator to the e-graph and return an `Id`. Note that
  we need not take any special care about "emission order" here,
  unlike when lowering, because the act of creating a node and adding
  it to the e-graph is essentially (if not in fact) a no-op on the
  data structure: the `Id` is just a hash-consed representation of a
  new node, and it can be ignored if not needed. Only the "union"
  operation (between given `Id` and returned `Id`) is semantically a
  mutation of the program, and this is done by the driver loop, not
  from within ISLE rules.
  
Then we write a top-level constructor as an entry point `(decl rewrite
(Id) Id)` that, given an e-class ID, creates and returns another
e-class ID to which it should be made equivalent (or the same, if no
rewrite is possible). Our top-level driver will invoke this on updated
e-classes in the batched way as described in the egg paper.

### Revisiting Legalization

Once we have a rewrite framework from CLIF to CLIF, it becomes natural
to ask: can we write legalizations as ISLE rules? The answer should
likely be yes. The main extension this requires beyond the above is to
allow side-effecting operators (in general these are dangerous to
reason about as expressions, but allowing them in a "replace this op
for that one" sense could be acceptably safe), and having a notion of
legalized-away nodes during extraction so that we always pick the
post-legalization form. This should finally allow us to remove the
`simple_legalize` framework, i.e., the last vestiges of the old
rewrite-based Cranelift lowering approach.

## Overall Cranelift Structure and Migration Path

Given the above e-graph-based means of rewriting and optimizing CLIF,
the overall structure of the Cranelift compilation pipeline will be:


```plain
               +------+                 +--------+
(frontend) --> | CLIF | --> (egraph --> | egraph | --> (egraph extract/
               +------+      build)     +--------+      elaborate)
                                          |   ^            |
                                          |   |            |
                                          v   |            v
                                        (rewrite       +------+
                                         rules)        | CLIF |
                                                       +------+
                                                           |
                          +-------+                        |
           (regalloc) <-- | VCode | <----  (lowering  <----'
                |         +-------+         rules)
                v
           +---------+
           | machine |
           | code    |
           +---------+
```

Importantly, the e-graph-based optimization pass consumes and produces
the same representation, CLIF. This means that we can omit the pass
for non-optimizing compiler configurations, and we can use the same
tooling to examine the program before and after optimization, compare
its execution, etc. This reuse of tooling and universality of the CLIF
representation is very important for practical reasons.

Eventually, we *could* short-circuit several steps in this flow for
efficiency:

- Some frontends could build an e-graph directly. We could either do
  this with a special mode in `cranelift-wasm` (for example), or
  perhaps, we could make it largely transparent behind the
  `InstBuilder` API. This would avoid the cost of building one data
  structure then transcribing it into another.
  
- Likewise, some backends could lower into VCode from the e-graph
  directly. Given that our rewrite rules will also be written in ISLE
  with extractor bindings, this feels very close-at-hand already. (In
  fact, earlier iterations of the ideas in this RFC did actually work
  this way.) The main complication is working out how to linearize the
  code, i.e. do the job of the extraction (cycle removal) and scoped
  elaboration passes described here, without actually building the
  CLIF output to be lowered. If we could somehow unify these
  algorithms with the existing backward lowering pass, we could save
  significant memory allocation and traffic.
  
  For even more advanced compilation, perhaps lowering rules could
  also operate directly with "multi-extractor" semantics, so we could
  allow instruction selection to see several possible views of the
  program (all e-nodes in a given e-class). This means that we no
  longer need a decoupled "cost model" in order to do extraction: we
  can just directly let the ISA-specific rule prioritization pick the
  cheapest representation by virtue of matching simpler instructions
  first.

Both of the above ideas are significantly more advanced than what we
want to do at first, and we anticipate (though we need to evaluate to
be sure) that the performance of the roundtrip through CLIF will be
adequate enough for us to start simple.

## Open Questions

The main open questions are:

1. How will e-graph-based optimization affect compile time? No
   practical production compiler, to our knowledge, has yet been built
   with an e-egraph-based representation at the core of its
   optimizer. We believe it can be practical based on our initial
   experimentation and our development of the scoped-elaboration
   algorithm, but constant factors are important, and we should be
   sure that we are not handicapping our compiler performance in any
   really significant way compared to handwritten optimization passes.
   
2. How well will the e-graph-based interleaving of rewrite rules
   improve upon separate handwritten passes with a carefully-chosen
   pass schedule? In other words, do we solve the pass ordering
   problem by composing at the per-rule-application granularity in the
   unified framework? Literature on e-graphs suggests that we can, but
   we will want to examine real examples once we have a reasonable
   start on the body of rules.

3. How well will the multi-extractor approach to adapting ISLE work?
   Or alternately, should we use egg's built-in rule engine somehow?
