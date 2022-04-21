# Summary

This RFC proposes a way to handle flexible vector types as specified at: https://github.com/WebAssembly/flexible-vectors

[summary]: #summary

The proposal is to introduce new dynamically-sized vector types into Cranelift, that:
- Enable dynamic vector type creation using existing fixed vector types and a dynamic scaling factor.
- The dynamic types, 'dt', express a vector with a lane type and shape, but a target-defined scaling factor.
- Space as been allocated in ir::Type for concrete definitions of these new types.
- The dynamic scaling factor is a global value which is defined by the target.
- We currently only support scaling factors which are compile-time constants.

# Motivation
[motivation]: #motivation

Flexible vectors are likely coming to WebAssembly so we should support the current spec. This is current path forward to support vectors that are wider than 128-bits.
Rust is also currently starting to use LLVM's ScalableVectorType and, as a target backend, Cranelift could support those directly with a dynamic vector type.

# Proposal
[proposal]: #proposal

The following proposal includes changes to the type system, adding specific entities for dynamic stack slots as well as specific instructions that take those entities as operands.

We can add dynamic vector types by modifying existing structs to hold an extra bit of information to represent the dynamic nature.

## Type System
- The new types do not report themselves as vectors, so ty.is\_vector() = false, but are explicitly reported via ty.is\_dynamic\_vector().
- is\_vector is also renamed to is\_sized\_vector to avoid ambiguity.
- The TypeSet, ValueTypeSet and TypeSetBuilder structs gains an extra NumSet to specify a minimum number of dynamic lanes.
- At the encoding level, space has been taken from the special types range to allow for the new types. Special types now occupy 0x01-0x2f and everything else has moved to fill the space, with dynamic types occupying the end of the range 0x80-0xff.
- These changes allow the usual polymorphic vector operations to be automatically built for the new set of dynamic types.

## IR Entities and Function Changes

A new global value is introduced `dyn_scale` which is parameterized by a base vector type. This global value can then be used to create dynamic types, such as `dt0 = i32x4*gv0`.

DynamicTypes are created and held like other IR entities, with the function holding a PrimaryMap\<DynamicType, DynamicTypeData\>. The DynamicTypeData holds the base vector type along with the GlobalValue which is the scaling factor.

A new entity is added to the IR, the DynamicStackSlot, and the Function will hold these separately from the existing StackSlot entities, also providing two APIs to create them:
- create\_sized\_stack\_slot
- create\_dynamic\_stack\_slot

Keeping two vectors enables us to continue to use each entity's index as it's slot identifier, and allows us to place the entities in different positions in the frame. It also enables the backends to easily track which slots are dynamic, and which are not. DynamicStackSlots are defined differently to existing StackSlots as they are defined with a DynamicType instead of a size, e.g. `dss0 = explicit_dynamic_slot dt0`

## Instructions
Three new instructions are added at the IR level to use the new DynamicStackSlot entity:
- DynamicStackAddr
- DynamicStackLoad
- DynamicStackStore

The primary difference between these operations and their existing counterparts is that they only take a DynamicStackSlot operand, without a byte offset.

DynamicVectorScale is the other instruction introduced, and this enables the materialization of a `dyn_scale` value when used by `globalvalue`.

## ABI Layer Changes

- The method stack_stackslot_addr is renamed to sized\_stackslot\_addr.
- The method dynamic\_stackslots\_addr is introduced.

A key challenge to supporting these new types is that the register allocator expects to be given an constant value for the size of a spill slot, for a given register class. So, the current expectation is that the backends will continue to provide a fixed number, potentially larger that they currently do. A new method on the TargetIsa trait, `vector_scale`, which returns the largest number of bytes for a given dynamic IR type. This is used by the ABI layer to cache the sizes of all the used dynamic types, the largest of which is used for the spillslot size. The size returned by the Isa is also used to calculate the dynamic stackslot offsets, just as is done for the existing stack slots. This means that the frame layout changes are minimal, just with the dynamic slots appended after the fixed size slots.

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
//!                              | spill slots               |
//!                              | (accessed via nominal SP) |
//!                              |          ...              |
//!                              | stack slots               |
//!                              | dynamic stack slots       |
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

The main change here is the introduction of dynamically created types, using an existing vector type as a base and a scaling factor represented by a global value. Using a global value fits with clif IR in that we have a value which is not allowed to change during the execution of the function. The alternative is to add types which have an implicit scaling factor which could make verification more complicated, or impossible.

The new vector types also aren't comparable to the existing vector types, which means that no existing paths can accidentally try to treat them as such.

It's possible that the hardware vector length will be fixed, so one alternative would be to generate IR with fixed widths using information from the backend. The one advantage is that we'd not have to add dynamic types in the IR at all. But there are two main disadvantages with this approach:
- In an ahead-of-time setting, Cranelift would not be able to take advantage of larger vectors where the width is implementation defined. It would be possible to target architecture extensions such as Intel's AVX2 and AVX-512, which have static sizes, but not for architectures like Arm's SVE.
- It is also currently undecided whether the flexible vector specification will include operations to set the vector length during program execution, so we shouldn't design out this possibility.

This doesn't mean that a backend can't select a fixed width during code generation, if desired. The current simd-128 implementations would be able to map the dynamic types directly to their current operations and we could also add a legalization layer for backends which only want to support simd-128, or another fixed size.

# Open questions
[open-questions]: #open-questions

- How will regalloc2 handle a new vector type and/or potential register aliasing? And will dynamic spill slots be possible?
- What behaviour would the interpreter have? I would expect it to default to the existing simd-128 semantics.
- Testing is also an issue, is it reasonable to assume that function under (run)test neither take or return dynamic vectors? If so, how should the result values be defined and checked against? I have currently implemented an instruction, extract\_vector, which takes a dynamic vector and an immediate which provides an index to a 128-bit sub-vector. Together with passing scalars as function parameters and splatting them into dynamic vectors, it allows simple testing of lane-wise operations.
