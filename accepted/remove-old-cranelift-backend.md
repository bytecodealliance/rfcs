# Summary
[summary]: #summary

This RFC proposes to remove the old x86-64 backend from Cranelift, and
subsequently clean up various bits of the IR, other compiler data
structures, and metaprogramming / codegen infrastructure that are only
used by old-style backends. The RFC represents two main decision
points: (i) whether it is now time to remove the old backend, and (ii)
what else in the codebase we will now be able to remove as a
consequence of this action.

# Motivation
[motivation]: #motivation

In RFC #10, we proposed switching the default Cranelift backend to the
new implementation, based on the `MachInst` framework. This new
framework is also in use by the aarch64 backend, which was natively
developed for the new APIs, and the recently-added s390x backend as
well.

To our knowledge, no significant consumers of Cranelift or Wasmtime
are still relying on the old backend, and it has been included as a
non-default option for two releases (0.27.x and 0.28.0) now. All
ongoing development effort is targetted to the new backends.

Alongside this, retaining the old backend as an option has several
ongoing costs. First, because we have decided thus far to ensure it
continues to pass tests, it imposes additional requirements on any new
features we might add. While we have sometimes adopted a more
fine-grained attitude of "this won't work on the old backend, and
that's OK" (see: newer Wasm instructions), it is still a decision that
has to be made in each case.

Second, and more significantly, the old backend is the sole factor
that keeps a number of other pieces of infrastructure alive. The core
IR data structures contain some abstractions that are relevant only
for the old backend, and this occasionally causes confusion. For
example, someone looking for information on stackslots, or regalloc
results, or code-layout information, will not find the information on
the CLIF after compilation is done: the new backends produce the
information in a separate IR (VCode). There is thus a cognitive
overhead involved in maintaining "old and deprecated" vs. "new and
spuported" status in one's head for bits of the compiler, and a
significant source of confusion for newcomers in particular.

# Proposal
[proposal]: #proposal

We thus propose to (i) remove the old x86 backend, making the x64
backend based on the `MachInst` framework the only supported option
for x86-64 in the future; and (ii) then performing "dead-code
elimination" as far as it will take us.

## Part 1: Deciding to Remove the Old Backend

The main question here is whether it is now the proper time to remove
the backend. In #10, we suggested maintaining its functionality while
"mak[ing] several releases with the new backend as default". We have
now done so for two releases (0.27.x and 0.28). We can also directly
consider several known users of Cranelift:

* Wasmtime: transitioned to new backend in
  bytecodealliance/wasmtime#2718. Feature flag to continue to use old
  backend.

* Lucet: transitioned to new backend in
  bytecodealliance/lucet#646. Feature flag to continue to use old
  backend.
  
* cg\_clif: transitioend to new backend in
  bjorn3/rustc_codegen_cranelift#1127 and removed ability to use old
  backend.
  
* Firefox/SpiderMonkey: most up-to-date integration (Baldrdash) used
  new backend only.
  
* VeriWasm: updated to support new x64 backend in PlSysSec/veriwasm#2.

Question 1: Are there any other known use-cases that remain on the old
backend?

Question 2: Is there any functionality in the old backend that we have
not yet adequately replicated in the new backend?

Question 3: given the above, is it acceptable to remove the old
backend?

This RFC proposes answering "yes" to Question 3 above, contingent on
receiving no answers to Question 1 or Question 2 that would change our
path.
  
## Part 2: Logistics

There are several steps that we can take, in order, to remove the old
backend and then carry out some clean-up work afterward.

Much of this work, especially the work to snip out the backend itself
and replace legalizations where needed, has already been drafted by
@bjorn3 in bytecodealliance/wasmtime#3009 (thanks!). This RFC's goal
is to gain consensus on a process around merging this work, and
outline the steps to carry it through with the appropriate cleanup
afterward.

1. Remove the `BackendVariant::Legacy` enum option. This is an
   API-breaking change that will force embedders who were explicitly
   selecting the old backend to see that the old backend is no longer
   available.
   
2. Remove the `old-x86-backend` Cargo build flag.

3. Remove the x86 backend itself: recipes and encodings in
   `cranelift/codegen/isa/x86/`.

4. Remove any remaining backend-specific CDSL / meta-crate code except
   for that which remains necessary. We believe this should include at
   least register definitions and platform-specific legalizations. We
   will need to replace some of the legalizations that the new backend
   relies on with handwritten versions in the `simple_legalize`
   framework.
   
5. Remove old code in the rest of the compiler.

   - Support for generating unwind info from the old backend's
     compilation result.
   - Support for generating debuginfo from the old backend's
     compilation result.
   - Compiler components only used when compiling with the old
     backend:
     - The register allocator.
     - The ABI legalization code.
     - The branch relaxation and binary emission pipeline.
   - Compiler data structures that are no longer used:
     - `encodings`, `locations`, `entry_diversions`, `offsets`,
       `jt_offsets`, prologue and epilogue info, etc., on
       `ir::Function`.
     - Any code that still generates/maintains/uses any of the above
       can and should be removed as well.

6. Begin to consider how the pipeline could be simplified in other
   ways now that some constraints are gone.
   - CodeSink: return machine code in some format more similar to the
     `MachBuffer`'s output, i.e., a single monolithic buffer rather
     than the `put1`/`put2`/... fine-grained API calling into the
     embedder repeatedly?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

As described above, this is the end of a long journey of refactoring
and transition to a cleaner design; there is no reason to keep the old
backend around once we've migrated all use-cases away from it and are
no longer spending effort maintaining it.

# Open questions
[open-questions]: #open-questions

1. Are there any significant users of the old backend that we have missed?

2. Is there any functionality in the old backend that we have not yet
   adequately replicated in the new backend?
