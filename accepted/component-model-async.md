# Summary
[summary]: #summary

Add support for the Component Model [Async
ABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Async.md)
to `wasm-tools`, `wit-bindgen`, and `wasmtime`.

# Motivation
[motivation]: #motivation

Although WASIp1 and WASIp2 both support multiplexing concurrent operations via
`poll_oneoff` and `wasi:io/poll`, respectively, neither of them support
_composable_ concurrency.  For example: composing two WASIp2 components, each of
which needs to do concurrent I/O, is not practical in general since only one of
them can call `wasi:io/poll#poll` at a time.

To address that limitation, WASIp3 is expected to rely on ABI extensions to the
[Component Model](https://github.com/WebAssembly/component-model/) supporting
async imports and exports, as well as built-in `stream`, `future`, and `error`
types.  These extensions make asynchronous composition as easy for application
developers as synchronous composition.  In addition, they neatly avoid the
["function coloring
problem"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
by allowing async-lowered imports to call sync-lifted exports and vice-versa.

In addition to solving the problem of composable concurrency, this approach
removes the need for WASIp2's `wasi:io` package, which has proven problematic
for virtualization.  In practice, virtualizing any interface that uses `wasi:io`
requires virtualizing all of them.  In the new design, I/O is handled via
built-in component model types which need not (and cannot) be virtualized,
thereby avoiding the issue.

Finally, the new async lower and lift ABI allows guest binding generators such
as those in `wit-bindgen` to produce more idiomatic code -- especially for
languages with `async`/`await`-style concurrency like JavaScript/TypeScript,
Python, C#, and Rust.  Application developers can simply import and export
`async` functions, leaving the details of inter-component task management to the
host.  Languages like Go, which rely on stackful coroutines, will now be able to
support concurrency natively, without requiring an
[Asyncify](https://emscripten.org/docs/porting/asyncify.html)-style
transformation.

## Relationship with the [`async-imports` RFC](./async-imports.md)

The earlier `async-imports` RFC proposed adding native `async` support to
Wasmtime, allowing embedders to call Wasm functions asynchronously such that a
Wasm call to an async host function may suspend the calling fiber and allow the
embedder to do other work concurrently.  That proposal has since been
implemented and has proven quite valuable for interoperating with
e.g. `tokio`-based servers.

However, the scope of that RFC is limited to making host->Wasm and Wasm->host
calls asynchronous only from the _host's_ perspective; from the guest's
perspective those calls are purely synchronous.  This proposal will build upon
that work to support calls which are asynchronous and concurrent from _both_ the
host's and guest's perspectives.  This means introducing a third way to call
Wasm functions, as well as a third way to add host functions to a
`wasmtime::component::LinkerInstance`, described below.

# Proposal
[proposal]: #proposal

I propose adding support to `wasm-tools` (e.g. `wasmparser`, `wasm-encoder`,
`wit-component`, etc.), `wit-bindgen`, and `wasmtime` for the new canonical ABI
options and built-ins described at a high level
[here](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Async.md)
and in more detail
[here](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md)
(look for the 🔀 icon, which indicates the new async features).  Note that, as
of this writing, the `stream`, `future`, and `error` features are not yet
included in the spec, but that work is in progress and expected to land soon.

Initially, this support will be gated by a disabled-by-default, experimental
feature flag (e.g. `component-async`).  Once the specification and
implementation have reached an appropriate level of maturity and test coverage,
we can consider enabling the feature by default.

For reference, I've created a
[prototype](https://github.com/dicej/component-async-demo) based on temporary
forks of `wasm-tools`, `wit-bindgen`, and `wasmtime`, demonstrating the
feasibility of supporting these features within the existing architectures of
each project.  That repository includes high-level, end-to-end tests of each
major feature, along with a proof-of-concept implementation of the
`wasi:http@0.3.0-draft` proposal.  Note that, as of this writing, the prototype
mostly predates the specification in the `component-model` repo and thus differs
from it regarding various details.  I'll be reconciling the implementation with
the spec as part of the process of upstreaming this work.  Also note that the
prototype uses a slightly different public API than is proposed here; the latter
represents my latest thinking and may be further modified based on feedback.

Regarding testing: I propose starting with a minimal suite of smoke tests as
part of each initial PR given the experimental nature of this feature and the
likelyhood of spec changes as we gather feedback on the design and
implementation.  As things stabilize, we will add more tests (including fuzz
tests) and make sure we're fully covered prior to enabling the feature by
default.

## `wasm-tools`

The changes required to `wasm-tools` are generally straightforward additions:

- Encoding, parsing, and validating the new canonical ABI options (`async` and
  `callback`) and functions (`task.backpressure`, `task.return`, `task.wait`,
  `stream.new`, etc.)
- `wit-component` support for "componentizing" a module which uses the above
- `wit-parser` additions to represent `stream`s, `future`s, and `error`s.  Note
  that `wit-parser` already has minimal support for representing `stream`s and
  `future`s, but those representations reflect an older, now-obsolete proposal,
  so we'll need to make some updates.
- `wasmprinter` and `wat` support for WAT<->binary conversion
- `wasm-smith`/`wit-smith` support for test case generation (although this might
  not be covered in the initial PR)

A couple items to note for `wit-component` and `wit-parser`:

- As of this writing,
  [BuildTargets.md](https://github.com/WebAssembly/component-model/pull/378)
  does not yet specify name mangling conventions for async features.  We'll need
  to add new functions to the `NameMangling` trait defined in `validation.rs` to
  support parsing the new async function names, but since there's no standard
  for them yet, we'll provide only stub implementations for the `Standard`
  implementation, meaning users will need to use the `Legacy` implementation
  until a standard is defined.
- From a subtyping perspective the `stream<T>` and `future<T>` types are
  invariant with respect to `T`.  However, since they are anonymous types which
  can appear anywhere in a WIT file, we need some way for bindings generators to
  refer unambiguously to each such type when generating calls to
  e.g. `stream.new`, `stream.drop`, etc.  I propose to handle this by
  identifying such types using a name mangling that includes the name of a WIT
  function in which the type appears (either as part of a parameter type or as
  part of the return type) along with an ordinal index indicating where the type
  appears when walking the types recursively left-to-right in depth-first order.
    - For example, given the WIT function `foo: func(x: future<future<u32>>, y:
      u32) -> stream<u8>`, the import `[future-new-0]foo` would indicate a call
      to `future.new` for the type `future<u32>`.  Likewise, `[future-new-1]foo`
      would indicate a call to `future.new` for `future<future<u32>>`, and
      `[stream-new-2]foo` would indicate a call to `stream.new` for `stream<u8>`.

## `wit-bindgen`

As a first step, we'll add support for these new features to the Rust binding
generator (macro and CLI) only, with `todo!()` stubs for other languages.  Based
on user interest and resources, we can incrementally add support for other
languages over time.

For the Rust generator, we'll add an `async` option to the `generate!` macro
(and a corresponding CLI option) to specify which imports and/or exports should
be generated as `async` functions, e.g.:

```rust
/// Specified which imports and/or exports should be async
///
/// Functions not explicitly listed here will be generated as synchronous bindings.
#[derive(Default, Debug, Clone)]
pub struct AsyncConfig {
    /// List of fully-qualified import names (e.g. `wasi:http/types@0.3.0-draft#[static]body.finish`)
    imports: HashSet<String>,
    /// List of fully-qualified export names (e.g. `wasi:http/handler@0.3.0-draft#handle`)
    exports: HashSet<String>,
}
```

Note that the `nonblocking` function attribute proposal should allow the user to
avoid configuring this explicitly.

In addition, we'll bundle an `async_support` module containing the following:

- Low-level (`#[doc(hidden)]`) support functions and types for managing tasks,
  callbacks, etc., invoked from generated async code
- Helper types, `impl`s, and functions for creating `future`s and `stream`s,
  spawning tasks, blocking via `task.wait`, and interoperating with the
  `futures` crate (e.g. `Stream<Vec<T>>` and `Sink<Vec<T>>` `impl`s for the read
  and write ends of a `stream<T>`, respectively).
  
## `wasmtime`

:tada: This is the fun one! :tada:  To summarize, I propose to add the following:

- A new `wasmtime::component::LinkerInstance::func_wrap_concurrent` function for
  registering host functions which are concurrent from _both_ the host and guest
  perspectives.  c.f. the existing `func_wrap` (synchronous for both the host
  and guest) and `func_wrap_async` (synchronous for the guest; asynchronous for
  the host) functions.
- A new `wasmtime::component::Func::call_concurrent` function (and likewise for
  `TypedFunc`) for calling Wasm functions concurrently with the same instance.
  c.f. the existing `call` (synchronous, requires a unique `Store` reference)
  and `call_async` (asynchronous, but returns a future which requires a unique
  `Store` reference, so can't be run concurrently with other calls to the same
  instance).
- `wasmtime::component::bindgen!` support for generating bindings based on
  `func_wrap_concurrent` and `call_concurrent`.
- Support for the new canonical options (`async`, `callback`) and built-in
  functions (`task.return`, `task.wait`, `stream.new`, etc.)
    - This includes fused adapter support for `async->async`, `async->sync`, and
      `sync->async` inter-component calls, extending the existing `sync->sync`
      support.
    - This also includes support for creating, suspending, and resuming fibers
      as necessary to support `async->sync` calls correctly, as well as
      providing backpressure as appropriate.
- New public APIs for creating and using `stream`s, `future`s, and `error`s,
  including passing them to and from Wasm guests.
  
Note that, with the `component-async` feature enabled, Wasmtime must be
prepared to handle async-lifted exports and async-lowered imports even if the
embedder is _not_ using the new `func_wrap_concurrent` or `call_concurrent`
APIs.  In other words, components using any combination of ABI options must be
run correctly regardless of the call style the embedder uses.  For example, an
async-lifted Wasm export called using the synchronous `Func::call` function must
work.  Likewise, an async-lowered Wasm import implemented via
`Linker::func_wrap` or `Linker::func_wrap_async` must also work, but note
neither of those methods allow guest-level concurrency, so they would only make
sense in cases where the host can complete the task without blocking or
`await`ing.

For the above reasons, I propose that, once the `component-async` feature is
enabled by default, we _deprecate_ `func_wrap_async` and modify the `bindgen!`
macro to stop using it by default (but still allow opting in to it via an option
during the deprecation period).  Rationale: a host function which needs to
`await` must account for the possibility the it has been called via an
async-lowered import and thus should not block the host _or_ the guest.
Conversely, a host function which is always non-blocking might as well use plain
old `func_wrap`.

On the other hand, there's no particular reason to deprecate `call_async`, which
may be more convenient than `call_concurrent` when the embedder only needs to
call one guest export at a time.  Unlike `func_wrap_async`, `call_async` cannot
inhibit the _guest's_ concurrency, so it's fine to use regardless of whether the
guest export is lifted sync or async.

The following sections provide more detailed descriptions of the items
summarized above.

## `LinkerInstance::func_wrap_concurrent`

This allows the caller to register host functions with the `LinkerInstance` such
that multiple calls to such functions can run concurrently.  This isn't possible
with the existing `func_wrap_async` method because it takes a function which
returns a future that owns a unique reference to the `Store`, meaning the
`Store` can't be used for anything else until the future resolves.

Ideally, we'd have a way to thread a `StoreContextMut<T>` through an arbitrary
`Future` such that it has access to the `Store` _only_ while being polled
(i.e. between, but not across, `await` points).  However, there's currently no
way to express that in async Rust, so we make do with a more awkward scheme:

```rust
    /// Accepts a function which receives a `StoreContextMut<T>` and the
    /// parameters received from the guest and returns a `Future` which does
    /// _not_ have access to the store.  The output of that `Future` is another
    /// function which gets access to the store again before producing the final
    /// result.
    pub fn func_wrap_concurrent<Params, Return, F, N, FN>(&mut self, name: &str, f: F) -> Result<()>
    where
        N: FnOnce(StoreContextMut<T>) -> Result<Return> + 'static,
        FN: Future<Output = N> + Send + Sync + 'static,
        F: Fn(StoreContextMut<T>, Params) -> FN + Send + Sync + 'static,
        Params: ComponentNamedList + Lift + 'static,
        Return: ComponentNamedList + Lower + Send + Sync + 'static,
```

In short, each function registered using `func_wrap_concurrent` gets access to
the `Store` twice: once before doing any concurrent operations (i.e. before
`await`ing) and once afterward.  This allows multiple calls to proceed
concurrently without any one of them monopolizing the store.  In practice, this
results in some ugly signatures and bodies when implementing such functions,
unfortunately:

```rust
    /// Implements `wasi:http@0.3.0-draft/types#[static]body.finish`
    fn finish(
        mut store: StoreContextMut<'_, Self::BodyData>,
        this: Resource<Body>,
    ) -> impl Future<
        Output = impl FnOnce(
            StoreContextMut<'_, Self::BodyData>,
        )
            -> wasmtime::Result<Result<Option<Resource<Fields>>, ErrorCode>>
                     + 'static,
    > + Send
           + Sync
           + 'static {
        let trailers = (|| {
            let trailers = store.data_mut().table().delete(this)?.trailers;
            trailers
                .map(|v| v.receive(store.as_context_mut()))
                .transpose()
        })();
        async move {
            let trailers = match trailers {
                Ok(Some(trailers)) => Ok(if let Ok(trailers) = trailers.await {
                    trailers
                } else {
                    None
                }),
                Ok(None) => Ok(None),
                Err(e) => Err(e),
            };

            component::for_any(move |_| Ok(Ok(trailers?)))
        }
    }
```

## `Func::call_concurrent`

This allows the host to call more than one Wasm export from the same instance
concurrently.  Analogously to `func_wrap_concurrent`, it returns a `Promise`
object which does _not_ require exclusive access to the `Store` in order to make
progress.  Instead, either a single `Promise` or a collection of them may be
converted into a `Future` which, when polled, allows _any_ outstanding `Promise`
for that `Store` (whether specified as input to the `Future` or not) to make
progress.

```rust
impl Func {
...
    /// Invoke this function with the specified `params`, returning a `Promise`
    /// that resolves to a tuple of the results and another `Promise`
    /// representing the completion of the task.
    pub fn call_concurrent(
        &self,
        params: &[Val],
    ) -> Promise<Result<(Vec<Val>, Promise<Result<()>>)>> { ... }
...
}

/// Represents result of a `Func::call_concurrent` invocation
///
/// Unlike a `Future`, instances of this type may make progress without being
/// polled directly.
struct Promise<T> { ... }

impl<T> Promise<T> {
    /// Convert this instance into a `Future` which may be `await`ed until the
    /// `Promise` resolves.
    ///
    /// Note that this may allow other, unrelated `Promise`s to also make
    /// progress.
    fn into_future<'a, U>(
        self,
        store: StoreContextMut<'a, U>,
    ) -> impl Future<Output = T> + Send + 'a
    where
        T: Send,
    { ... }

    /// Convert the specified `Promise`s into a `Future` which may be `await`ed
    /// until one or more of the `Promise`s resolve.
    ///
    /// Note that this may allow other, unrelated `Promise`s to also make
    /// progress.
    fn select<'a, U>(
        store: StoreContextMut<'a, U>,
        promises: Vec<Promise<T>>,
    ) -> impl Future<Output = (Vec<T>, Vec<Promise<T>>)> + Send + 'a
    where
        T: Send,
    { ... }

    /// Map the output of the specified promise to another type.
    fn map<V>(self, fun: impl FnOnce(T) -> V) -> Promise<V> { ... }
}
```


## `component::bindgen!` updates

I propose to modify the host binding generator to optionally generate bindings
that use `func_wrap_concurrent` rather than `func_wrap_async` for async imports.
This mode of operation would be added in three stages:

1. Add a boolean `concurrent_imports` option to the
`wasmtime::component::bindgen!`  macro which defaults to `false`.
2. Once the `component-async` feature is enabled by default, change the default
to `true`.
3. Remove the option entirely once `func_wrap_async` support is removed.

Likewise, we can add a `concurrent_exports` option which, when set to `true`,
tells the generator to generate `call_concurrent` calls rather than `call_async`
calls.  As noted earlier, there's no reason to deprecate `call_async`, though,
and we can choose a default value for `concurrent_exports` based on user
feedback.

## Support for new canonical options and built-in functions

A lot of this work is mechanical: plumbing the options and built-in calls
through various layers of Wasmtime code and processing them according to the
specification:

- Translating the `wasmparser` representations into several stages of
  `wasmtime-environ` representations
- Generating fused adapters as necessary for each combination of import/export
  used (sync/sync, sync/async, async/sync, async/async)
- Generating trampolines in `wasmtime-cranelift`
- Handling built-in function calls at runtime in the `wasmtime` crate

The most interesting part is the last one.  Wasmtime must now manage a top-level
event loop kicked off by the first call to an async-lifted export, alternating
between waiting for host tasks to make progress and delivering events to guest
tasks (either by calling callback functions or resuming fibers).

`async->sync` calls are particularly interesting, since the host must either
call the sync export in a new fiber or queue the call for later if one is
already in progress for that instance.  If and when that fiber makes a blocking
host call, it must be suspended so that the caller can continue to execute.
Fortunately, `wasmtime-fiber` already provides a robust API for managing fibers,
and Wasmtime's existing `async` support demonstrates how to use it effectively
in the context of an async runtime.  The main innovation required here is to
support multiple fibers per top-level component instance and safely pass access
to the `Store` from one to another as appropriate.

Another interesting case is connecting a `{stream|future}.send` call with a
corresponding `{stream|future}.receive` call.  Ideally, we'd handle this with a
fused adapter, except in this case we can't know statically which component
instance will be the receiver (and thus what string encoding it is using, etc.).
When a component sends data to the write end of a `stream`, the receiver end
could be held by any other component instance under the same top-level instance.
In general, if there are N components and M distinct payload types we don't want
to generate (N^2)*M fused adapters and select among them dynamically at runtime;
nor do we want to do bunch of static analysis to wittle down the number of
possible cases.  Instead, we analyze the payload type ahead of time, using an
optimized, `memcpy`-based host call for "flat" payloads (i.e. those without
pointers or handles) and a slower host call for other types based on the
`wasmtime::component::Val` dynamic API.

Finally, note that `stream`s and `future`s can generally be treated as `own`ed
resource handles, except that they are generic with respect to their payload
type and cannot be borrowed.  `error`s, on the other hand, have
reference-counted, immutable value semantics, meaning a given `error` can be
passed to another component but remain available in the original component until
explicitly dropped.
      
## Host APIs for creating, using, and sharing `stream`s, `future`s, and `error`s

Ideally, the public API for `stream`s and `future`s would look a lot like the
[futures::channel::mpsc](https://docs.rs/futures/latest/futures/channel/mpsc/index.html)
and
[futures::channel::oneshot](https://docs.rs/futures/latest/futures/channel/oneshot/index.html)
APIs, respectively.  Or perhaps
[AsyncRead](https://docs.rs/futures/latest/futures/io/trait.AsyncRead.html) and
[AsyncWrite](https://docs.rs/futures/latest/futures/io/trait.AsyncWrite.html)
APIs for the special case of `stream<u8>`.  However, we have the same challenge
here as with `call_concurrent`: each of these types need a `StoreContextMut<T>`
to do their job, i.e. lifting from and lowering to Wasm linear memory (which may
involve `cabi_realloc` calls, etc.).  Therefore, as with `call_concurrent`, we
use the `Promise` type described above to achieve `Store`-friendly concurrency.

```rust
pub struct FutureSender<T> { ... }

impl<T> FutureSender<T> {
    pub fn send(self, value: T) -> Promise<Result<()>>
    where
        T: func::Lower + Send + Sync + 'static,
    { ... }

    pub fn close<U, S: AsContextMut<Data = U>>(self, store: S) -> Result<()> { ... }
}

pub struct FutureReceiver<T> { ... }

impl<T> FutureReceiver<T> {
    pub fn receive(self) -> Promise<Result<Option<T>>>
    where
        T: func::Lift + Sync + Send + 'static,
    { ... }

    pub fn close<U, S: AsContextMut<Data = U>>(self, store: S) -> Result<()> { ... }
}

pub fn future<T, U, S: AsContextMut<Data = U>>(
    store: S,
) -> Result<(FutureSender<T>, FutureReceiver<T>)> { ... }

pub struct StreamSender<T> { ... }

impl<T> StreamSender<T> {
    pub fn send(&mut self, values: Vec<T>) -> Promise<Result<()>>
    where
        T: func::Lower + Send + Sync + 'static,
    { ... }

    pub fn close<U, S: AsContextMut<Data = U>>(self, store: S) -> Result<()> { ... }
}

pub struct StreamReceiver<T> { ... }

impl<T> StreamReceiver<T> {
    pub fn receive(&mut self) -> Promise<Result<Option<Vec<T>>>
    where
        T: func::Lift + Sync + Send + 'static,
    { ... }

    pub fn close<U, S: AsContextMut<Data = U>>(self, store: S) -> Result<()> { ... }
}

pub fn stream<T, U, S: AsContextMut<Data = U>>(
    store: S,
) -> Result<(StreamSender<T>, StreamReceiver<T>)> { ... }
```

The `error` API, on the other hand, is quite simple since you can't do anything
with it except pass it around:

```rust
pub struct Error { ... }

pub fn error<U, S: AsContextMut<Data = U>>(store: S) -> Result<Error> { ... }
```

We expect that `error`s will eventually have `anyhow`-style functionality
(diagnostic messages, backtraces, downcast-able payloads, etc.), but for now
they're just opaque handles.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- As noted above, the problem of interleaving unique `Store` references with
  `Future`s and the rest of Rust's `async` machinery results in some pretty
  awkward APIs.  I've tried to minimize the changes to Wasmtime as much as
  possible, but perhaps this calls for a refactor of the `wasmtime` crate and
  API with both guest-level concurrency and thread-based parallelism in mind --
  one which enables a more ergonomic async/stream/future API.
- The host APIs for `stream`s described above deals with `Vec<T>` types.  We
  could optimize that API and implementation by using slices instead to avoid
  allocation overhead while still minimizing copies.  Ideally, we'd provide an
  API for true zero-copy I/O via e.g. `io_uring` such that bytes are delivered
  directly to a guest's linear memory, but that will require more thought and
  can always be added leter.
  
In general, the details of the implementation can always be refined once we've
established a sane public API.

# Open questions
[open-questions]: #open-questions

- Is there a more ergonomic approach to interleaving unique `Store` references
  with Rust's `async` machinery without sacrificing type and memory safety?
- While implementing the
  [prototype](https://github.com/dicej/component-async-demo), I found that
  calling `func_wrap_concurrent` required using `Box<dyn ...>` trait objects to
  make the borrow checker happy, and that in turn required adding a `'static`
  bound to the `T` type argument to `StoreContextMut<T>`, which is unfortunate.
  I _suspect_ this is a limitation in the Rust type system that could be fixed,
  but we should find out for sure.
- What's the most expedient way to test callback-less, async-lifted exports?  Go
  would be the first likely candidate to support this, but I'm not sure how much
  work that would be.  LLVM-based toolchains use a shadow stack and will need to
  be modified to support multiple shadow stacks.  Meanwhile, we can always use
  handwritten WAT.
- Async function calls and calls to certain canonical built-in functions (such
  as `task.return`) require converting between `*mut dyn VMStore` and
  `StoreContextMut<T>`, which is [highly unsafe for multiple
  reasons](https://github.com/bytecodealliance/wasmtime/blob/d88aeda8a09a88386b8953fc3f285fffcd141c59/crates/wasmtime/src/runtime/store/context.rs#L22-L38).
  One of those reasons is that converting from a (non-aliasable) `&mut` to an
  (aliasable) `*mut` and then back to a (non-aliasable) `&mut` invalidates the
  original `&mut` -- i.e. it must no longer exist or be used.  In my prototype,
  I tried to address this by making every function that takes a
  `StoreContextMut<T>` and possibly does such a `&mut->*mut->&mut` conversion
  also _return_ a `StoreContextMut<T>` (in addtion to anything else it needs to
  return).  The caller is then required to use the returned `StoreContextMut<T>`
  rather than the original one (i.e. no reborrow tricks).  Is that sufficient to
  ensure soundness?  Are there more robust options?
