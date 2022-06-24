# Summary
[summary]: #summary
 
This proposal adds an *instance allocator* abstraction to the Wasmtime runtime that will allow for custom allocation of instances and related data such as WebAssembly memories and tables.

In addition to this abstraction, this proposal will outline an implementation of a *pooling instance allocator* that will manage a pool of available instances that are allocated in advance.
 
# Motivation
[motivation]: #motivation
 
Merging Lucet's features that enable very fast module instantiation into Wasmtime is the primary motivation for this proposal.

Therefore it is important to understand how Lucet accomplishes this today.

## Lucet regions
 
Lucet uses a concept called a [`Region`](https://github.com/bytecodealliance/lucet/blob/main/lucet-runtime/lucet-runtime-internals/src/region.rs) that manages all of the host process address space needed to represent many concurrently running instances.

This enables the very fast instance creation and destruction required by high-load services that use WebAssembly to handle requests because allocations are kept to a minimum and the memory backing an instance may be reused between instatiations from unrelated modules.
 
The default Lucet `Region` implementation is `MmapRegion` that manages a fixed-capacity set of equal-sized *slots*, where each **used** slot represents a running module instance.
 
A slot is a contiguous region of memory containing:
 
* The instance's data (execution context, region allocation data, signal handler, etc).
* The instance's singular memory ("heap") with guard pages.
* The instance's execution stack used for yielding execution to the host.
* The instance's globals as an array of 8-byte values (`v128` isn't supported).
* The per-instance signal stack.
 
As each slot needs to be of equal size, upfront limits must be set on the region to allocate the memory for each slot as a contiguous block.
 
In Lucet those limits are:
 
* The maximum size of an instance's memory (`heap_memory_size`).
* The maximum size of an instance's address space (`heap_address_space_size`).
* The size of the instance's execution stack (`stack_size`).
* The size of the stack reserved for host calls (`hostcall_reservation`).
* The maximum size of an instance's globals in bytes (`globals_size`).
* The size of the instance's signal stack (`signal_stack_size`).
 
Lucet doesn't support the reference types proposal that expanded the WebAssembly instruction set to include table mutation instructions as well as allowing for multiple tables.  As a result, tables are not stored in a slot; instead tables are stored in a read-only data section of the ELF executable representing a `lucetc`-compiled WebAssembly module.
 
The multi-memory and bulk memory proposals are also not supported in Lucet, so an instantiated module may only have a single memory that is actively (i.e. upon instantiation) initialized.

## Lucet's `uffd` feature

Lucet supports a `uffd` feature that, when enabled for Linux, will offer an alternative implementation of `Region` called `UffdRegion` that uses Linux's `userfaultfd` system call to handle page faults through an entirely contiguous region of memory.

This feature has several advantages over the default `MmapRegion` implementation:

* `mprotect` does not need to be called to change the protection level of memory pages upon instance creation and destruction.  This reduces lock contention in the kernel's virtual memory manager and Lucet will therefore track which pages are accessible itself.

* Active data segment initialization does not need to occur at module instantiation time because Lucet can handle faults to accessed pages and initialize the data per-page when accessed for the first time.

The `uffd` feature is very important to high-load scenarios where system call and kernel lock contention would impact service throughput.

## AOT compilation

Lucet uses an ahead-of-time (AOT) compiler to translate WebAssembly modules into ELF shared objects files that can be loaded into a host process.

Wasmtime does not yet support AOT compilation, although an implementation is planned.

This proposal attempts to integrate Lucet's memory management features with Wasmtime today with the expectation that modules are currently just-in-time (JIT) compiled, but both will be supported in the future.
 
# Proposal
[proposal]: #proposal

This proposal outlines changes to the `wasmtime_runtime` and `wasmtime` crates needed to implement a pooling instance allocator with similar functionality to Lucet's region implementation.

## Changes to the `wasmtime_runtime` crate

### The `InstanceAllocationRequest` struct

```rust
/// Represents a request for a new runtime instance.
pub struct InstanceAllocationRequest<'a> {
   /// The module being instantiated.
   pub module: Arc<Module>,

   /// The finished (JIT) functions for the module.
   pub finished_functions: &'a PrimaryMap<DefinedFuncIndex, *mut [VMFunctionBody]>,

   /// The imports to use for the instantiation.
   pub imports: Imports<'a>,

   /// A callback for looking up shared signature indexes.
   pub lookup_shared_signature: &'a dyn Fn(SignatureIndex) -> VMSharedSignatureIndex,

   /// The host state to associate with the instance.
   pub host_state: Box<dyn Any>,

   /// The pointer to the VM interrupts structure to use for the instance.
   pub interrupts: *const VMInterrupts,

   /// The pointer to the reference activations table to use for the instance.
   pub externref_activations_table: *mut VMExternRefActivationsTable,

   /// The pointer to the stack map registry to use for the instance.
   pub stack_map_registry: *mut StackMapRegistry,
}
```

This is simply an encapsulation of the current arguments to `InstanceHandle::new` that will be passed to an instance allocator to create a new instance.

### The `InstanceAllocator` trait

```rust
/// Represents a runtime instance allocator.
///
/// # Safety
///
/// This trait is unsafe as it requires knowledge of Wasmtime's runtime internals to implement correctly.
pub unsafe trait InstanceAllocator: Send + Sync {
   /// Validates a module translation.
   ///
   /// This is used to ensure a module being compiled is supported by the instance allocator.
   fn validate_module(&self, translation: &ModuleTranslation) -> Result<(), String>;

   /// Adjusts the tunables prior to creation of any JIT compiler.
   ///
   /// This method allows the instance allocator control over tunables passed to a `wasmtime_jit::Compiler`.
   fn adjust_tunables(&self, tunables: &mut wasmtime_environ::Tunables);

   /// Allocates an instance for the given allocation request.
   ///
   /// # Safety
   ///
   /// This method is not inherently unsafe, but care must be made to ensure
   /// pointers passed in the allocation request outlive the returned instance.
   unsafe fn allocate(
      &self,
      req: InstanceAllocationRequest,
   ) -> Result<InstanceHandle, InstantiationError>;

   /// Finishes the instantiation process started by an instance allocator.
   ///
   /// # Safety
   ///
   /// This method is only safe to call immediately after an instance has been allocated.
   unsafe fn initialize(
      &self,
      handle: &InstanceHandle,
      is_bulk_memory: bool,
      data_initializers: &Arc<[OwnedDataInitializer]>,
   ) -> Result<(), InstantiationError>;

   /// Deallocates a previously allocated instance.
   ///
   /// # Safety
   ///
   /// This function is unsafe because there are no guarantees that the given handle
   /// is the only owner of the underlying instance to deallocate.
   ///
   /// Use extreme care when deallocating an instance so that there are no dangling instance pointers.
   unsafe fn deallocate(&self, handle: &InstanceHandle);
}
```

As `Instance` is private to the `wasmtime_runtime` crate, this trait must return `InstanceHandle` and instance allocator implementations 
must reside in `wasmtime_runtime`.

This trait will be implemented by two types in the runtime: an on-demand instance allocator that allocates instances based on Wasmtime's 
current implementation and a new *pooling* instance allocator that will function more like Lucet's region implementations.

The current implementation of `InstanceHandle::new` will be moved into the on-demand instance allocator, but some common instantiation 
implementation will be shared between the two allocators.

### The `OnDemandInstanceAllocator` struct

```rust
/// Represents the on-demand instance allocator.
pub struct OnDemandInstanceAllocator { ... }

impl OnDemandInstanceAllocator {
   /// Creates a new default instance allocator.
   pub fn new(mem_creator: Option<Arc<dyn RuntimeMemoryCreator>>) -> Self { ... }
}

impl InstanceAllocator for OnDemandInstanceAllocator { ... }
```

This type will be used to encapsulate the current implementation in the Wasmtime runtime for allocating instances, where allocating an instance will also allocate the related resources and deallocating an instance will free the related resources.

For backwards compatibility, a `RuntimeMemoryCreator` can be used to control how host memory is allocated for any linear memories.

### The `ModuleLimits` struct
 
```rust
/// Represents the limits placed on a module for compiling with the pooling instance allocator.
#[derive(Copy, Clone)]
pub struct ModuleLimits {
    /// The maximum number of imported functions for a module (default is 1000).
    pub imported_functions: usize,

    /// The maximum number of imported tables for a module (default is 0).
    pub imported_tables: usize,

    /// The maximum number of imported memories for a module (default is 0).
    pub imported_memories: usize,

    /// The maximum number of imported globals for a module (default is 0).
    pub imported_globals: usize,

    /// The maximum number of defined types for a module (default is 100).
    pub types: usize,

    /// The maximum number of defined functions for a module (default is 10000).
    pub functions: usize,

    /// The maximum number of defined tables for a module (default is 1).
    pub tables: usize,

    /// The maximum number of defined memories for a module (default is 1).
    pub memories: usize,

    /// The maximum number of defined globals for a module (default is 10).
    pub globals: usize,

    /// The maximum table elements for any table defined in a module (default is 10000).
    ///
    /// If a table's minimum element limit is greater than this value, the module will
    /// fail to compile.
    ///
    /// If a table's maximum element limit is unbounded or greater than this value,
    /// the maximum will be `table_elements` for the purpose of any `table.grow` instruction.
    pub table_elements: usize,

    /// The maximum number of pages for any memory defined in a module (default is 160).
    ///
    /// The default of 160 means at most 10 MiB of host memory may be committed for each instance.
    ///
    /// If a memory's minimum page limit is greater than this value, the module will
    /// fail to compile.
    ///
    /// If a memory's maximum page limit is unbounded or greater than this value,
    /// the maximum will be `memory_pages` for the purpose of any `memory.grow` instruction.
    ///
    /// This value cannot exceed any address space limits placed on instances.
    pub memory_pages: usize,
}

impl Default for ModuleLimits {
    fn default() -> Self {
        // See doc comments for `ModuleLimits` for these default values
        Self {
            imported_functions: 1000,
            imported_tables: 0,
            imported_memories: 0,
            imported_globals: 0,
            types: 100,
            functions: 10000,
            tables: 1,
            memories: 1,
            globals: 10,
            table_elements: 10000,
            memory_pages: 160,
        }
    }
}
```

The values in this structure are used to determine how much address space gets reserved by the pooling instance allocator.

Modules will be validated against these limits after translation but before JIT compilation to ensure the module can be instantiated.

### The `InstanceLimits` struct

```rust
/// Represents the limits placed on instances by the pooling instance allocator.
#[derive(Copy, Clone)]
pub struct InstanceLimits {
    /// The maximum number of concurrent instances supported (default is 1000).
    pub count: usize,

    /// The maximum reserved host address space size to use for each instance in bytes.
    ///
    /// Note: this value has important performance ramifications.
    ///
    /// On 64-bit platforms, the default for this value will be 6 GiB.  A value of less than 4 GiB will
    /// force runtime bounds checking for memory accesses and thus will negatively impact performance.
    /// Any value above 4 GiB will start eliding bounds checks provided the `offset` of the memory access is
    /// less than (`address_space_size` - 4 GiB).  A value of 8 GiB will completely elide *all* bounds
    /// checks; consequently, 8 GiB will be the maximum supported value. The default of 6 GiB reserves
    /// less host address space for each instance, but a memory access with an offset above 2 GiB will incur
    /// runtime bounds checks.
    ///
    /// On 32-bit platforms, the default for this value will be 10 MiB. A 32-bit host has very limited address
    /// space to reserve for a lot of concurrent instances.  As a result, runtime bounds checking will be used
    /// for all memory accesses.  For better runtime performance, a 64-bit host is recommended.
    pub address_space_size: usize,
}

impl Default for InstanceLimits {
    fn default() -> Self {
        // See doc comments for `InstanceLimits` for these default values
        Self {
            count: 1000,
            #[cfg(target_pointer_width = "32")]
            address_space_size: 0xA00000,
            #[cfg(target_pointer_width = "64")]
            address_space_size: 0x180000000,
        }
    }
}
```

The `address_space_size` value is used to calculate the total address space needed by the pooling instance allocator.

The remaining limits are enforced at module instantiation time.

### The `PoolingAllocationStrategy` enum

```rust
/// The allocation strategy to use for the pooling instance allocator.
#[derive(Clone)]
pub enum PoolingAllocationStrategy {
    /// Allocate from the next available instance.
    NextAvailable,
    /// Allocate from a random available instance.
    Random,
}

impl Default for PoolingAllocationStrategy {
    fn default() -> Self {
        Self::NextAvailable
    }
}
```

This enumeration controls how the pooling instance allocator locates free objects in the pools of instances.

The intention is to reduce the predictability of host process address space dedicated to instances, not unlike address space layout randomization (ASLR).

This is similar to Lucet's `AllocStrategy` for regions.

### The `PoolingInstanceAllocator` struct

```rust
/// Represents a pooling instance allocator.
pub struct PoolingInstanceAllocator { ... }

impl PoolingInstanceAllocator {
   /// Creates a new pooling instance allocator with the given strategy and limits.
   pub fn new(
      strategy: PoolingAllocationStrategy,
      module_limits: ModuleLimits,
      instance_limits: InstanceLimits,
   ) -> Result<Self, String> { ... }
}

impl InstanceAllocator for PoolingInstanceAllocator { ... }
```

This type is responsible for reserving large, contiguous regions of address space that can be used to quickly allocate instances in 
Wasmtime.

The implementation will use a free list to track the available instances that can be handed out by the allocator.

When an instance is deallocated, it will be returned to a free list.

### The `userfault` feature

Like Lucet's `uffd` feature, the `userfault` feature will control whether or not the pooling instance allocator will handle page faults in userspace.  The implementation will be heavily based on Lucet's.

When enabled, the Linux implementation of `PoolingInstanceAllocator` will create a thread that will monitor a file descriptor that has been created with the `userfaultfd` system call.

Page faults will be handled according to where they occur in the memory managed by the pooling instance allocator.

Implementations for other platforms, such as Windows, might be required in the future.

### The `Instance` struct

To minimize allocations, `Instance` will be modified to use `Vec` instead of boxed slices for storing the instance's lists of memories and tables.  This will allow the pooling allocator to reuse capacity from previous instance allocations as needed.

To further reduce the allocations performed when creating an `Instance`, two bit arrays will be used instead of the two `HashMap` that are storing the "not-yet-dropped" passive data and element segments.

The `table.init` and `memory.init` implementations can first check the bit array for a dropped segment (i.e. bit at index is 1) and treat the segment as empty; otherwise, source the segment directly from the instance's associated `Module`.  The `data.drop` and `elem.drop` implementations therefore become a bit set operation.

Additional changes are required to enable storing the instance's tables in the memory reserved by the pooling instance allocator rather than as a `Vec` per table, but this proposal considers that an implementation detail.

## Changes to the `wasmtime` crate

### The `userfault` feature

This feature will forward to the runtime's `userfault` feature to enable userfault handling in the runtime's pooling instance allocator.

### The `InstanceAllocationStrategy` enumeration
```rust
// Represents the module instance allocation strategy to use.
#[derive(Clone)]
pub enum InstanceAllocationStrategy {
    /// The on-demand instance allocation strategy.
    ///
    /// Resources related to a module instance are allocated at instantiation time and
    /// immediately deallocated when the `Store` referencing the instance is dropped.
    ///
    /// This is the default allocation strategy for Wasmtime.
    OnDemand,
    /// The pooling instance allocation strategy.
    ///
    /// A pool of resources is created in advance and module instantiation reuses resources
    /// from the pool. Resources are returned to the pool when the `Store` referencing the instance
    /// is dropped.
    Pooling {
        /// The allocation strategy to use.
        strategy: PoolingAllocationStrategy,
        /// The module limits to use.
        module_limits: ModuleLimits,
        /// The instance limits to use.
        instance_limits: InstanceLimits,
    },
}

impl Default for InstanceAllocationStrategy { ... }
```

### The `Config` struct

This proposal adds the following method to `Config` for setting the instance allocation strategy to use:

```rust
/// Sets the instance allocation strategy to use.
pub fn with_instance_allocation_strategy(
   &mut self,
   strategy: InstanceAllocationStrategy,
) -> Result<&mut Self>;
```

All module instances created from the configuration will use the given strategy.

### The `MemoryCreator` trait

Today, users can implement the `MemoryCreator` trait to control how linear memories are allocated when Wasmtime instantiates a module or a user creates a host `Memory` object.

To enable custom host memory management, users call `with_host_memory` on the `Config` used to create an `Engine`.  For backwards compatibility, this interface will not change.

Only the default instance allocation strategy will use memory creator for creating linear memories for new module instances.

Host `Memory` objects will continue to use the given memory creator for allocating memory.

### Re-exported types

The following types will be re-exported from `wasmtime_runtime`:

* `ModuleLimits`
* `InstanceLimits`
* `PoolingAllocationStrategy`

## API Example

```rust
let mut config = Config::new();
config.with_instance_allocation_strategy(InstanceAllocationStrategy::Pooling {
   strategy: PoolingAllocationStrategy::Random,
   module_limits: ModuleLimits::default(),
   instance_limits: InstanceLimits::default(),
})?;

let engine = Engine::new(&config);
let module = Module::new(&engine, r#"(module (func (export "run")) )"#);

let store = Store::new(&engine);
let instance = Instance::new(&store, &module, &[])?;

let run = instance.get_func("run").unwrap().get0::<()>()?;

run()?;
```

All instances created with `engine` will use the configured instance allocator.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This proposal attempts to work within the confines of the existing `wasmtime_runtime` crate.

Ideally the pooling instance allocator would be split off into its own crate, but the visibility of the `Instance` structure in the runtime prevents this.
 
# Open questions
[open-questions]: #open-questions

* Should the pooling instance allocator implementation be behind a cargo feature or in a separate crate?

* The defaults for `PoolingLimits` are inspired by Lucet's default limits.  Are these sufficient for general use?
 
* Will there be a need for representing these types with the other language bindings (e.g. C, C#, Golang, Python, etc.)?
