# Summary
[summary]: #summary

This document proposes to create a pair of tools to support running components
using any compatible WebAssembly runtime, including those with little or no
native component support.  These tools could be used either as an alternative to
providing native support for components or as a temporary polyfill while native
support is being built for that runtime.

# Motivation
[motivation]: #motivation

Implementing the complete Component Model specification is a non-trivial task
which entails significant up-front effort as well as ongoing maintenance.  [Jco]
has proven useful as a polyfill for JS-embedded Wasm runtimes which don't yet
have native component support, but it entails a performance penalty and doesn't
support stand-alone Wasm runtimes.

On the other hand, though Wasmtime has performant, native support for
components, that code cannot easily be reused for other runtimes.  In addition,
that implementation significantly expands the trusted compute base which must be
audited for correct and secure behavior beyond what is needed for core Wasm.  It
is closely integrated with the unsafe internals of the runtime, requiring
specialized knowledge and care to maintain and modify.

Ideally, a component implementation would provide the best qualities of both of
those implementations, while addressing or side-stepping their weaknesses:

- Portable to arbitrary runtimes (JS-embedded or standalone)
- Performant
- Secure, e.g. doing as much work (and allocation) as practical in sandboxed
  guest code, minimizing the TCB
- Maintainable without specialized knowledge of the internals of a particular
  runtime
- Compatible with embedded and/or memory-limited scenarios

[Jco]: https://github.com/bytecodealliance/jco

# Proposal
[proposal]: #proposal

This proposal includes four things:

- A `lower-component` tool which takes a component as input and "lowers" it into
  a core module
- A `host-wit-bindgen` tool and runtime library which takes a WIT world and
  produces code for a chosen target language to instantiate a
  `lower-component`-generated module, invoke its exports, and satisfy its
  imports
- A host<->guest C API representing the intrinsics and imports which the
  `host-wit-bindgen`-generated code and runtime must provide for a given WIT
  world, plus the exports which the `lower-component`-generated module can be
  expected to provide
- An embedding C API representing the operations which a host runtime must provide to
  enable instantiation, invocation, access to memories and globals, etc. for the
  `host-wit-bindgen`-generated code and runtime library to use

## `lower-component`

The job of this tool is to take an arbitrary component as input and "lower" it
into a core module which may be run using an arbitrary Wasm runtime, provided it
supports the `host-wit-bindgen`-compatible embedding API.  This tool could
either be used ahead-of-time or just prior to instantiation.

In general, a component may include a composition of more than one subcomponent,
each instantiation of which may require its own memory and table.  In that case,
the output module will use multiple memories and tables and include generated
adapter code to "fuse" the imports of one component to the exports of another,
handling validation, cross-memory copies, etc., just as Wasmtime's FACT does
today.

In addition to the generated "fused adapter" code, the output module will
include component model runtime code, separately compiled from Rust source,
which handles, among other things:

- table management for resource and waitable values
- guest-to-guest stream and future I/O
- task and thread bookkeeping

That runtime code will itself make use of intrinsic functions imported from the
host in order to do things only the host can do, including:

- creating, suspending, and resuming fibers
- tracking host-owned and borrowed resources, streams, futures, waitable sets,
  and tasks
- reading from and writing to streams and futures where one end is owned by the
  host
- collecting error backtraces

### Multiply-instantiated Modules

One challenge with lowering arbitrary components is that a component may
instantiate the same module more than once.  In that case, we have three options:

- Reject the component
- Generate an output module with duplicate copies of each function in a
  multiply-instantiated module, one per instance.
    - Note that leaf functions which do not use memory or globals can be reused
      without duplication.
    - This could lead to significant bloat for "batteries-included" guest
      languages like Python which do not have dead-code-elimination.
- Generate multiple output modules, plus metadata indicating how to instantiate
  and link them together.
    - This would require specifying how that metadata is represented and how the
      whole combination of modules+metadata should be packaged.

Regarding the third option: one possibility would be to define a minimal subset
of the component model which only supports declaring, linking, and instantiating
core modules and use that to package the modules.
  
## Host<->Guest C API for Lowered Components

This API includes two parts:

- Imported functions called by the guest and provided by the host
- Exported functions called by the host and provided by the guest

### Guest->Host API

As mentioned above, modules produced using `lower-component` can't (yet) express
certain operations in core Wasm, and therefore must use intrinsics when:

- creating, suspending, and resuming fibers
- polling or waiting for one or more host events
- reading from and writing to streams and futures where one end is owned by the host
- generating stack traces for component-level errors

Fiber management could be expressed using the Stack Switching proposal, and
indeed `lower-component` will likely have an option to use those instructions,
but that proposal is not yet widely implemented, so we use intrinsics for
maximum portability.

Note that these intrinsics need not be implemented in C, nor a language with
native support for the Wasm C ABI; we simply use C as a way to represent the ABI
in a familiar, human-readable format.

Finally, note that the exact set of functions imported by a given lowered
component will depend on which intrinsics it actually needs, which stream and
future types are used in the world it targets, and which interfaces and
functions are imported by the world.  For example, imagine we've lowered a
component which targets the following WIT world:

```
package example:package;

interface foo {
  resource thing {
    constructor(v: u32);
    get: func() -> u32;
  }
  
  bar: async func(v: string, s: stream<u32>) -> stream<thing>;
}

world target {
  import foo;
  export foo;
}
```

The following is a sketch of the API imported by such a lowered component as a
set of C functions.  Here we assume that the component explicitly creates and
switches between threads, and thus must import intrinsics from the host to do
so.

Note that all of the following constants, types, and functions are (eventually)
intended to match the code a C binding generator would generate per the proposed
[Guest C ABI](https://github.com/WebAssembly/component-model/pull/378).


```c
// The types and functions in this code block are based on a subset of the
// intrinsics defined by the
// [Component Model ABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md),
// with modifications in some cases to represent e.g. memory and table indexes
// as runtime values rather than component-level declarations.

// Creates a new thread, initially in a "suspended" state.
//
// - `context`: The value to pass to the function in the new thread
// - `table`: The table in which to find the function
// - `func`: The function to call, of type `(func (param i32))`
//
// Returns the host-defined identifier for the newly-created thread.
__attribute__((__import_module__("env"), __import_name__("[thread.new]")))
uint32_t thread_new(void *context, uint32_t table, uint32_t func);

// Switch to the specified thread, suspending the current one.
//
// - `thread`: The identifier of the thread to which to switch
__attribute__((__import_module__("env"), __import_name__("[thread.switch-to]")))
void thread_switch_to(uint32_t thread);

// Retrieves the identifier for the currently-running thread.
__attribute__((__import_module__("env"), __import_name__("[thread.index]")))
uint32_t thread_index();

// Retrieves the zeroth thread-local context variable.
__attribute__((__import_module__("env"), __import_name__("[context[0].get]")))
uint32_t context0_get();

// Sets the zeroth thread-local context variable.
__attribute__((__import_module__("env"), __import_name__("[context[0].set]")))
void context0_set(uint32_t value);

// Creates a new `waitable-set`.
//
// Note that this would be used only for tracking host-visible waitables;
// internal, guest-only waitables would be managed without involving the host.
__attribute__((__import_module__("env"), __import_name__("[waitable-set.new]")))
uint32_t waitable_set_new();

// Adds the specified `waitable` to the specified `waitable-set` (or remove it from any set
// if zero).
//
// - `waitable`: The `waitable` to add or remove
// - `set`: The set to which to add the `waitable`, or zero if the `waitable` is
//   to be removed
__attribute__((__import_module__("env"), __import_name__("[waitable.join]")))
uint32_t waitable_join(uint32_t waitable, uint32_t set);

#define EVENT_NONE 0
#define EVENT_SUBTASK 1
#define EVENT_STREAM_READ 2
#define EVENT_STREAM_WRITE 3
#define EVENT_FUTURE_READ 4
#define EVENT_FUTURE_WRITE 5
#define EVENT_CANCELLED 6

// Represents the result of a call to `waitable-set.{wait,poll}`
//
// - `event`: One of the `EVENT_*` constants defined above
// - `waitable`: `waitable` to which the event pertains, if any
// - `payload`: Event-specific payload, if any
//
// Note that we use a currently-hypothetical `__multivalue_return__` attribute
// here to indicate that functions returning this type should compile to a core
// Wasm type of e.g. `(func ... (result i32 i32 i32))`.
__attribute__((__multivalue_return__))
typedef struct {
  uint32_t event;
  uint32_t waitable;
  uint32_t payload;
} wait_result_t;

#define COPY_RESULT_BLOCKED 0xFFFFFFFF
#define COPY_RESULT_COMPLETED 0
#define COPY_RESULT_DROPPED 1
#define COPY_RESULT_CANCELLED 2

// Blocks the calling fiber until the specified `waitable-set` has an event.
//
// - `set`: The `waitable-set` to wait for
__attribute__((__import_module__("env"), __import_name__("[waitable-set.wait]")))
wait_result_t waitable_set_wait(uint32_t set);

// Polls the specified `waitable-set` to determine if it has any pending events.
//
// - `set`: The `waitable-set` to poll
__attribute__((__import_module__("env"), __import_name__("[waitable-set.poll]")))
wait_result_t waitable_set_poll(uint32_t set);

// Represents the write- and read-ends of a `stream` or `future`.
__attribute__((__multivalue_return__))
typedef struct {
  uint32_t writer;
  uint32_t reader;
} writer_reader_pair_t;

// Constructs a new `stream<u32>`.
//
// Note that the guest would only call this when it intends to give the readable
// end to the host at some point.  It would _not_ be called for guest-to-guest
// streams (except to "upgrade" such a stream to one which can be passed to the
// host) since those are managed internally by the guest.
//
// Returns the (writer, reader) pair.
__attribute__((__import_module__("env"), __import_name__("[stream<u32>.new]")))
writer_reader_pair_t stream_u32_new();

// Constructs a new `stream<thing>`, where `thing` is the _imported_ resource.
//
// Returns the (writer, reader) pair.
__attribute__((__import_module__("env"), __import_name__("[stream<import example:package/foo#thing>.new]")))
writer_reader_pair_t stream_import_example_package_foo_thing_new();

// Constructs a new `stream<thing>`, where `thing` is the _exported_ resource.
//
// Returns the (writer, reader) pair.
__attribute__((__import_module__("env"), __import_name__("[stream<export example:package/foo#thing>.new]")))
writer_reader_pair_t stream_export_example_package_foo_thing_new();

// Represents the result of a stream read or write.
//
// - `result`: One of the `COPY_RESULT_*` constants defined above
// - `count`: The number of items copied, if any
__attribute__((__multivalue_return__))
typedef struct {
  uint32_t result;
  size_t count;
} result_and_count_t;

// Reads from a `stream<u32>` whose write end is owned by the host.
//
// - `stream`: The identifier of the stream from which to read
// - `memory`: The memory in which the buffer resides.
// - `buffer`: The buffer to receive the items
// - `length`: The maximum number of items which may be received
//
// The return value indicates the result of the operation in the same format as
// the return value of the `stream.read` CM intrinsic.
__attribute__((__import_module__("env"), __import_name__("[stream<u32>.read]")))
result_and_count_t stream_u32_read(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for writing to a stream whose read end is owned by the host.
__attribute__((__import_module__("env"), __import_name__("[stream<u32>.write]")))
result_and_count_t stream_u32_write(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for `stream<thing>`, where `thing` is the _imported_ resource
// type.
__attribute__((__import_module__("env"), __import_name__("[stream<import example:package/foo#thing>.read]")))
result_and_count_t stream_import_example_package_foo_thing_read(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for writing.
__attribute__((__import_module__("env"), __import_name__("[stream<import example:package/foo#thing>.write]")))
result_and_count_t stream_import_example_package_foo_thing_write(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for `stream<thing>`, where `thing` is the _exported_ resource
// type.
__attribute__((__import_module__("env"), __import_name__("[stream<export example:package/foo#thing>.read]")))
result_and_count_t stream_export_example_package_foo_thing_read(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for writing.
__attribute__((__import_module__("env"), __import_name__("[stream<export example:package/foo#thing>.write]")))
result_and_count_t stream_export_example_package_foo_thing_write(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// Creates a new host-defined handle for an object of the _exported_ resource
// type `thing`.
//
// Note that the guest would only call this when it intends to pass the handle
// to the host (by own or borrow) at some point.  It would _not_ be used for
// guest-to-guest calls, in which case everything would be managed internally
// by the guest.
__attribute__(( __import_module__("[export]example:package/foo"), __import_name__("[resource.new]thing")))
extern uint32_t __wasm_import_exports_foo_bar_my_interface_thing_new(uint32_t v);

// Returns the guest-side representation for a handle to an object of the
// _exported_ resource type `thing`.
__attribute__((__import_module__("[export]example:package/foo"), __import_name__("[resource.rep]thing")))
extern uint32_t __wasm_import_exports_foo_bar_my_interface_thing_rep(uint32_t);

// Drops the handle to an object of the _exported_ resource type `thing`, making
// it invisible to the host.
__attribute__((__import_module__("[export]example:package/foo"), __import_name__("[resource.drop]thing")))
extern void __wasm_import_exports_foo_bar_my_interface_thing_drop(int32_t handle);

// Returns a value from the _exported_ `bar` function.
//
// - `value`: The value to return.
__attribute__((__import_module__("[export]example:package/foo"), __import_name__("[task.return]bar")))
void __wasm_export_exports_foo_bar_my_interface_bar__task_return(uint32_t value);

// Confirms cancellation of an earlier call to the _exported_ `bar` function.
__attribute__((__import_module__("[export]example:package/foo"), __import_name__("[task.cancel]bar")))
void __wasm_export_exports_foo_bar_my_interface_bar__task_cancel();
```

```c
// The types and functions in this code block represent normal, non-intrinsic
// imports of the target world.

// Constructor for the _imported_ resource `thing`.
//
// - `v`: The constructor's `u32` parameter
//
// Returns the host-defined identifier for the newly-created object.
__attribute__((__import_module__("example:package/foo"), __import_name__("[constructor]thing")))
uint32_t import_example_package_foo_constructor_thing(uint32_t v);

// `get` method for the _imported_ resource `thing`.
//
// - `handle`: The identifier of the object
//
// Returns the `u32` result.
__attribute__((__import_module__("example:package/foo"), __import_name__("[method]thing.get")))
uint32_t import_example_package_foo_thing_get(uint32_t handle);

// Drops a borrow or own handle to an instance of the _imported_ resource `thing`.
//
// - `handle`: The identifier of the object
__attribute__((__import_module__("example:package/foo"), __import_name__("[resource-drop]thing")))
void import_example_package_foo_thing_drop(uint32_t handle);

#define TASK_STATUS_STARTING 0
#define TASK_STATUS_STARTED 1
#define TASK_STATUS_RETURNED 2
#define TASK_STATUS_START_CANCELLED 3
#define TASK_STATUS_RETURN_CANCELLED 4

// Represents the result of a call to an async-lowered function import.
//
// - `status`: One of the `TASK_STATUS_*` constants defined above
// - `task`: The host-defined identifier for the task, if `status < 2`
__attribute__((__multivalue_return__))
typedef struct {
  uint32_t status;
  uint32_t task;
} task_result_t;

// Imported `bar` function.
//
// - `memory`: The memory to which `v_ptr` and `return_ptr` pointers point
// - `v_ptr`: A pointer to the UTF-8-encoded string representing the `v` parameter
// - `v_len`: The length, in bytes of the encode string
// - `s`: `stream<u32>` parameter
// - `return_ptr`: A pointer to space reserved to receive the result `stream<thing>`
//
// Returns the result of the call as a `task_result_t`.
__attribute__((__import_module__("example:package/foo"), __import_name__("bar")))
task_result_t import_example_package_foo_bar(
  uint32_t memory, uint8_t *v_ptr, size_t v_len, uint32_t s, uint32_t *return_ptr
);
```

### Host->Guest API

This is the other half of the host<->guest C API, covering the functions which
`host-wit-bindgen` can expect the `lower-component`-generated module to export.

As with the Guest->Host API, the exact set of functions exported by a given
lowered component will depend on which resource, stream, and future types are
used by the world targeted by the component, as well as which interfaces and
functions are exported.  The following is a sketch of the API exported by the
hypothetical lowered component we presented in the previous section.

Again, all of the following constants, types, and functions are (eventually)
intended to match the imports a C binding generator would generate per the
proposed [Guest C ABI](https://github.com/WebAssembly/component-model/pull/378).

```c
// (Re)allocates from the specified guest memory.
//
// - `memory`: The index of the memory from which to allocate
// - `ptr`: The previous allocation, or `NULL`
// - `old_size`: The size of the previous allocation, if applicable
// - `align`: The minimum alignment of the new allocation
// - `new_size`: The minimum size of the new allocation
//
// Returns the new allocation, or traps on failure.
//
// Note that this function may become unnecessary once
// [Lazy Lowering](https://github.com/WebAssembly/component-model/issues/383)
// arrives.
__attribute__((__export_name__("cabi_realloc")))
void *cabi_realloc(
  uint32_t memory, void *ptr, size_t old_size, size_t align, size_t new_size
);

// Constructor for the _exported_ resource `thing`
//
// - `v`: The constructor's `u32` parameter
//
// Returns the identifier of the newly-constructed object.
__attribute__((__export_name__("example:package/foo#[constructor]thing")))
uint32_t example_package_foo_constructor_thing(uint32_t v);

// `get` method for the _exported_ resource `thing`
//
// - `handle`: The borrow handle to the target object
//
// Returns the `u32` result.
__attribute__((__export_name__("example:package/foo#[method]thing.get")))
uint32_t example_package_foo_thing_get(uint32_t handle);

// Disposes of an instnace of the _exported_ resource `thing`.
//
// - `handle`: The identifier of the object to be disposed of
__attribute__((__export_name__("example:package/foo#[dtor]thing")))
void example_package_foo_thing_dtor(uint32_t handle);

#define CALLBACK_CODE_EXIT 0
#define CALLBACK_CODE_YIELD 1
#define CALLBACK_CODE_WAIT 2

// Represents the result of a call to an async export.
//
// - `code`: One of the `CALLBACK_CODE_*` constants defined above
// - `waitable_set`: The `waitable-set` on which to wait if `code == CALLBACK_CODE_WAIT`
__attribute__((__multivalue_return__))
typedef struct {
  uint32_t code;
  uint32_t waitable_set;
} task_status_t;

// Exported `bar` function.
//
// - `memory`: The memory to which `v_ptr` and `return_ptr` pointers point
// - `v_ptr`: A pointer to the UTF-8-encoded string representing the `v` parameter
// - `v_len`: The length, in bytes of the encode string
// - `s`: `stream<u32>` parameter
// - `return_ptr`: A pointer to space reserved to receive the result `stream<thing>`
//
// Returns the status of the task.
__attribute__((__export_name__("[async-lift]example:package/foo#bar")))
task_status_t example_package_foo_bar(
  uint32_t memory, uint8_t *v_ptr, size_t v_len, uint32_t s, uint32_t *return_ptr
);

// Callback for exported `bar` function.
//
// - `event`: The event to be delivered
// - `waitable`: `waitable` to which the event pertains, if any
// - `payload`: Event-specific payload, if any
__attribute__((__export_name__("[callback][async-lift]example:package/foo#bar")))
task_status_t example_package_foo_bar_callback(
  uint32_t event, uint32_t waitable, uint32_t payload
);

```

## `host-wit-bindgen`

This tool takes as input a WIT world and produces source code for a given target
language which may be used to define component-level host functions, instantiate
`lower-component`-produced modules, and invoke its exports, etc.  It also
includes a runtime library containing reusable code for e.g. tracking waitables,
managing host<->guest stream and future I/O, etc.

Similar to `wit-bindgen`, this could be packaged either as a single tool
supporting multiple target languages or as separate tools, one per language.  In
either case, the generated code and runtime library would bottom out in calls to
the embedding C API described in the next section.

Alternatively, the functionality of `host-wit-bindgen`-generated code could be
provided by a library providing a general-purpose, dynamic API for creating
component values, defining host functions, and calling functions.  This would be
useful in scenarios where the shape of the component is not known ahead of time,
and/or the target language is already so dynamic that code generation is
redundant.

## Host C API for Embedder Bindings

In theory, `host-wit-bindgen` could support multiple front-ends (e.g. Rust,
Python, C#, Go, etc.) _and_ multiple back-ends (Wasmtime, WAMR, Wazero, JS,
etc.), but it's probably easier to define a runtime-agnostic API which each
runtime can implement to support the low-level operations required by
`host-wit-bindgen`-generated code.  Those operations include:

- creating a "store" in which one or more modules may be instantiated
- defining host functions
- instantiating a module
- calling a module's exports
- reading from and writing to a module's memories, tables, and globals
- creating, suspending, and resuming fibers

Given that a C API doesn't make sense in e.g. a web browser, this could be
mirrored as a JS API for use in JS-embedded runtimes.

```c
// Note that we disregard error handling (and, to some extent, efficiency)
// for simplicity in the following API sketch.

// Represents a runtime store in which one or more modules may be instantiated.
typedef struct {
  void *ptr;
} store_t;

// Creates a new store.
//
// - `data`: Application-specific data to be associated with the store
store_t store_new(void *data);

// Gets the application-specific data from the store.
//
// - `store`: The store from which the data should be retrieved
//
// Returns the associated data.
void *store_data(store_t store);

// Disposes of the specified store.
void store_drop(store_t store);

// Represents a linker to which host-defined functions may be added.
typedef struct {
  void *ptr;
} linker_t;

// Creates a new linker.
linker_t linker_new();

// Represents a core Wasm value.
typedef union {
  uint32_t u32;
  uint64_t u64;
  float f32;
  double f64;
} value_t;

// Represents a host-defined function.
//
// - `store`: The store to which the calling instance belongs
// - `instance`: The calling instance
// - `param_ptr`: A buffer containing the function parameters
// - `param_len`: The number of function parameters
// - `result_ptr`: A buffer to receive the function results
// - `result_len`: The capacity of the result buffer
typedef void (*host_function_t)(
  store_t store,
  instance_t instance,
  value_t *param_ptr,
  size_t param_len,
  value_t *result_ptr,
  size_t result_len
);

// Adds the specified function to the linker.
//
// - `linker`: The linker to which the function will be added
// - `module`: The name of the module from which the function may be imported
// - `name`: The name of the function to add
// - `func`: The function to add
void linker_add(linker_t linker, const char *module, const char *name, host_function_t func);

// Dispose of the specified linker.
void linker_drop(linker_t linker);

// Represents an instantiated and linked module.
typedef struct {
  void *ptr;
} instance_t;

// Instantiates the specified module.
//
// - `store`: The store in which the instance will be created
// - `linker`: The linker containing any host functions for the instance to import
// - `module`: The Wasm module to instantiate
instance_t instance_new(store_t store, linker_t linker, uint8_t *module);

// Calls an exported function in the specified instance
//
// - `store`: The store to which the instance belongs
// - `instance`: The instance in which the function is exported
// - `name`: The name of the export
// - `param_ptr`: A buffer containing the function parameters
// - `param_len`: The number of function parameters
// - `result_ptr`: A buffer to receive the function results
// - `result_len`: The capacity of the result buffer
void instance_call(
  store_t store,
  instance_t instance,
  const char *name,
  value_t *param_ptr,
  size_t param_len,
  value_t *result_ptr,
  size_t result_len
);

// Retrieves the value of the specified global variable.
//
// - `store`: The store to which the instance belongs
// - `instance`: The instance in which the global resides
// - `index`: The index of the global variable
value_t instance_get_global(store_t store, instance_t instance, uint32_t index);

// Sets the value of the specified global variable.
//
// - `store`: The store to which the instance belongs
// - `instance`: The instance in which the global resides
// - `index`: The index of the global variable
// - `value`: The value to set
void instance_set_global(store_t store, instance_t instance, uint32_t index, value_t value);

// Represents the result of an `instance_get_memory` call.
typedef struct {
  uint8_t *ptr;
  size_t len;
} memory_result_t;

// Retrieves a pointer to the specified memory.
//
// - `store`: The store to which the instance belongs
// - `instance`: The instance in which the memory resides
// - `index`: The index of the memory
memory_result_t instance_get_memory(store_t store, instance_t instance, uint32_t index);

// Represents a store-managed fiber.
typedef struct {
  void *ptr;
} fiber_t;

// Creates a new fiber.
//
// - `store`: The store in which the fiber will be created
// - `context`: Application-defined state to be passed to `func`
// - `func`: Function to call when the fiber is resumed for the first time
fiber_t fiber_new(store_t store, void *context, void (*func)(void*));

// Passes control to the specified fiber.
//
// - `store`: The store to which the fiber belongs
// - `fiber`: The fiber to resume
void fiber_resume(store_t store, fiber_t fiber);

// Suspends the currently running fiber.
void fiber_suspend();
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

See the `Motivation` section above for rationale.

At a high level, the alternative to creating a runtime-agnostic component model
implementation is to implement a native one for each runtime, possibly with some
parts factored out into reusable libraries.  This is the approach we've taken so
far with Wasmtime, although not necessarily the one we'd choose with the benefit
of hindsight.  In any case, a runtime-agnostic implementation would be useful as
a temporary polyfill for use in a given runtime until a native implementation is
complete.

## Prior art

There are already a few projects which polyfill the component model:

- [Jco](https://github.com/bytecodealliance/jco) for JS-embedded runtimes
- [Gravity](https://github.com/arcjet/gravity) for Wazero on Go
- [Meld](https://github.com/pulseengine/meld) for arbitrary runtimes

# Open questions
[open-questions]: #open-questions

- How to handle multiply-instantiated modules, and how common are the components
  which do that?
- Who should be responsible for type checking during host->guest and guest->host
  invocations?
    - If `host-wit-bindgen`, where in the flattened module will it find the type
      metadata it needs?
    - If `lower-component`, it could be harder to optimize (e.g. doing it on
      every call, whereas `host-wit-bindgen` could lean on the target language's
      static type guarantees, as `wasmtime-wit-bindgen` does today)
- Consider ways to improve the embedding API to make it misuse-resistant but
  still efficient.
    - For example, `instance_get_memory` and other functions could return a
      guard object representing exclusive access to the store which must be
      released prior to using that store for anything else.
