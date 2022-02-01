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

The following proposal includes changes to the type system, adding specific entities for sizeless stack slots as well as specific instructions that take those entities as operands.

We can add sizeless vector types by modifying existing structs to hold an extra bit of information to represent the sizeless nature.

## Type System
- The new types do not report themselves as vectors, so ty.is\_vector() = false, but are explicitly reported via ty.is\_sizeless\_vector().
- is\_vector is also renamed to is\_sized\_vector to avoid ambiguity.
- The TypeSet and ValueTypeSet structs gain a bool to represent whether the type is sizeless.
- TypeSetBuilder also gains a bool to control the building of those types.
- At the encoding level, a bit is used to represent whether the type is sizeless and this bit has been taken from special types range.
- These changes allow the usual polymorphic vector operations to be automatically built for the new set of sizeless types.

## IR Entities and Function Changes
A new entity is added to the IR, the SizelessStackSlot, and the Function will hold these separately from the existing StackSlot entities, also providing two APIs to create them:
- create\_sized\_stack\_slot
- create\_sizeless\_stack\_slot

Keeping two vectors enables us to continue to use each entity's index as it's slot identifier, and allows us to place the entities in different positions in the frame. It also enables the backends to easily track which slots are sizeless, and which are not.

## Instructions
Three new instructions are added at the IR level to use the new SizelessStackSlot entity:
- SizelessStackAddr
- SizelessStackLoad
- SizelessStackStore

The primary difference between these operations and their existing counterparts is that they only take a SizelessStackSlot operand, without a byte offset.

## ABI Layer Changes

- The method stack_stackslot_addr is renamed to sized\_stackslot\_addr.
- The method sizeless\_stackslots\_addr is introduced, which takes a vector\_scale parameter.
- get\_number\_of\_spillslots\_for\_value is also modified to take a vector\_scale parameter.

A key challenge to supporting these new types is that the register allocator expects to be given an constant value for the size of a spill slot, for a given register class. So, the current expectation is that the backends will continue to provide a fixed number, potentially larger that they currently do. This value is provided to the ABI layer via a new method on the TargetIsa trait, vector\_scale, which returns the largest number of bytes for a target vector register. This can then also be used to scale the index when calculating the address of a SizelessStackSlot, if a backend chooses a fixed sized during code generation.

With the notion of a sizeless stack slot possible, but not a sizeless spill slot, the proposed frame layout would look like the following:

```
//! ```plain
//!   (high address)
//!
//!                              +---------------------------+
//!                              |          ...              |
//!                              | stack args                |
//!                              | (accessed via FP)         |
//!                              +---------------------------+
//! SP at function entry ----->  | return address            |
//!                              +---------------------------+
//!                              |          ...              |
//!                              | clobbered callee-saves    |
//! unwind-frame base     ---->  | (pushed by prologue)      |
//!                              +---------------------------+
//! FP after prologue -------->  | FP (pushed by prologue)   |
//!                              +---------------------------+
//!                              | sizeless stack slots      |
//!                              | (accessed via FP)         |
//!                              |          ...              |
//!                              +---------------------------+
//!                              | spill slots               |
//!                              | (accessed via nominal SP) |
//!                              |          ...              |
//!                              | stack slots               |
//!                              | (accessed via nominal SP) |
//! nominal SP --------------->  | (alloc'd by prologue)     |
//! (SP at end of prologue)      +---------------------------+
//!                              | [alignment as needed]     |
//!                              |          ...              |
//!                              | args for call             |
//! SP before making a call -->  | (pushed at callsite)      |
//!                              +---------------------------+
//!
//!   (low address)
//! ```
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main design decision is to have a vector type that isn't comparable to the existing vector types, which means that no existing paths can accidently try to treat them as such. Although setting the minimum size of the opaque type to 128 bits still allows us to infer the minimum number of lanes for a given lane type. The flexible vector specification also provides specific operations for lane accesses and shuffles so these aren't operations that we have to handle with our existing operations.

It's possible that the hardware vector length will be fixed, so one alternative would be to generate IR with fixed widths using information from the backend. The one advantage is that we'd not have to add sizeless types in the IR at all. But there are two main disadvantages with this approach:
- In an ahead-of-time setting, Cranelift would not be able to take advantage of larger vectors where the width is implementation defined. It would be possible to target architecture extensions such as Intel's AVX2 and AVX-512, which have static sizes, but not for architectures like Arm's SVE.
- It is also currently undecided whether the flexible vector specification will include operations to set the vector length during program execution, so we shouldn't design out this possibility.

This doesn't mean that a backend can't select a fixed width during code generation, if desired. The current simd-128 implementations would be able to map the sizeless types directly to their current operations and we could also add a legalization layer for backends which only want to support simd-128, or another fixed size.

# Open questions
[open-questions]: #open-questions

- How will regalloc2 handle a new vector type and/or potential register aliasing? And will sizeless spill slots be possible?
- What behaviour would the interpreter have? I would expect it to default to the existing simd-128 semantics.
- Testing is also an issue, is it reasonable to assume that function under (run)test neither take or return sizeless vectors? If so, how should the result values be defined and checked against? I have currently implemented an instruction, extract\_vector, which takes a sizeless vector and an immediate which provides an index to a 128-bit sub-vector. Together with passing scalars as function parameters and splatting them into sizeless vectors, it allows simple testing of lane-wise operations.
