# Summary

This RFC proposes a way to handle flexible vector types as specified at: https://github.com/WebAssembly/flexible-vectors

[summary]: #summary

The proposal is to introduce new sizeless vector types into Cranelift, that:
- Express a vector, with a lane type and size, but a target-defined number of lanes.
- Are denoted by prefixing the lane type with 'sv' (sizeless vector): svi16, svi32, svf32, etc...
- Have a minimum width of 128-bits, meaning we can simply map to existing simd-128 implementations.

# Motivation
[motivation]: #motivation

Flexible vectors are likely coming to WebAssembly so we should support the current spec. This is current path forward to support vectors that are wider than 128-bits.
Rust is also currently starting to use LLVM's ScalableVectorType and, as a target backend, Cranelift could support those directly with a sizeless vector type.

# Proposal
[proposal]: #proposal

We can add sizeless vector types by modifying existing structs to hold an extra bit of information to represent the sizeless nature:

- The new types don't report themselves as vectors, so ty.is\_vector() = false, but are explicitly reported via ty.is\_sizeless\_vector().
- The TypeSet and ValueTypeSet structs gain a bool to represent whether the type is sizeless.
- TypeSetBuilder also gains a bool to control the building of those types.
- At the encoding level, a bit is used to represent whether the type is sizeless, this bit has been taken from the max number of vector lanes supported, so they'd be reduced to 128 from 256.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main design decision is to have a vector type that isn't comparable to the existing vector types, which means that no existing paths can accidently try to treat them as such. Although setting the minimum size of the opaque type to 128 bits still allows us to infer the minimum number of lanes for a given lane type. The flexible vector specification also provides specific operations for lane accesses and shuffles so these aren't operations that we have to handle with our existing operations.

It's possible that the hardware vector length will be fixed, so one alternative would be to generate IR with fixed widths using information from the backend. The one advantage is that we'd not have to add sizeless types in the IR at all. But there are two main disadvantages with this approach:
- In an ahead-of-time setting, Cranelift would not be able to take advantage of larger vectors where the width is implementation defined. It would be possible to target architecture extensions such as Intel's AVX2 and AVX-512, which have static sizes, but not for architectures like Arm's SVE.
- It is also currently undecided whether the flexible vector specification will include operations to set the vector length during program execution, so we shouldn't design out this possibility.

This doesn't mean that a backend can't select a fixed width during code generation, if desired. The current simd-128 implementations would be able to map the sizeless types directly to their current operations and we could also add a legalization layer for backends which only want to support simd-128, or another fixed size.

# Open questions
[open-questions]: #open-questions

- Does anyone care that Cranelift could only support a maximum of 128 vector lanes? The only problem I could imagine is if someone has a 128-lane vector of 1-bit bools...
- How will the register allocator (regalloc2..?) handle a new vector type and/or potential register aliasing?
- Are there parts of Cranelift, which aren't backend specific, that would need to handle these types? (is there generic stack handling or anything else data size specific...?)
- What behaviour would the interpreter have? I would expect it to default to the existing simd-128 semantics.
- Testing is also an issue, is it reasonable to assume that function under (run)test neither take or return sizeless vectors? If so, how should the result values be defined and checked against?
