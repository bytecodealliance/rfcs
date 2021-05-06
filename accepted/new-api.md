# Summary
[summary]: #summary

Overhaul the `wasmtime` crate's API to improve it along a number of vectors:

* Greatly improve the multithreading story in all languages (Rust, C, Go, ...),
  namely enabling safe usage of `wasmtime` in multithreaded server runtimes
  where wasm objects can migrate between threads.
* Leverage mutability in Rust to allow `&mut` access to user-defined state
  instead of requiring users to use interior mutability.
* Simplify memory management in the C API and embeddings.

The only major cost relative to today's API is that `ExternRef` will move to
atomic reference counting and will require `T: Send + Sync` on constructors.

# Motivation
[motivation]: #motivation

The `wasmtime` crate's API as-is today is heavily inspired by the [JS WebAssembly][js]
and the proposed [`wasm.h`] APIs. The principles of the Rust API are evident in
the `wasm.h` and `wasmtime.h` C APIs in addition to other language bindings
such as Python, Go, and .NET. While these APIs of `wasmtime` have stood
relatively well against the test of time, some cracks have been getting wider
over time to the point that they're becoming difficult to work around.

These are some key situations where the current API does not work well:

* Multithreaded WebAssembly runtimes have no safe recourse for using Wasmtime.
  While [there is documentation][multidoc] of how to do this it involves liberal
  usage of `unsafe` in Rust and is quite difficult to actually show that it's
  correct. Additionally language bindings like Go have to go to great lengths to
  respect the multithreaded requirements of the current API, often at the cost
  of performance.

* The Rust `wasmtime` crate API only ever provides shared (`&T`) access to host
  state in host functions. This means that interior mutability is pervasively
  used throughout embeddings of Wasmtime. Interior mutability is not the most
  idiomatic in Rust and has many hidden traps which can make it especially
  difficult for beginners to work with. This issue compounds the multithreaded
  issue above because typically used constructs like `Rc`, `RefCell`, and `Cell`
  all cause the Rust compiler to further complain about the `Send` and `Sync`
  properties of types.

* Management of memory in the API can sometimes be confusing. Additionally for C
  the management can be cumbersome since all objects are distinctly owned (e.g.
  `wasm_func_t` has a destructor). The confusing part arises from how you can
  create a host function, but when deleting that host function (either by
  garbage collecting it or calling its destructor) it doesn't actually delete
  the backing host memory. This is because all objects actually live in a
  "store" and deleting handles is just that, deleting a handle, not the object
  itself. This proves especially cumbersome for embeddings in safe languages
  who need to ensure proper deallocation of all objects. This again compounds
  with the multithreaded issue above where objects also have to be dropped on
  the same thread, which is not the default for languages like Go.

[js]: https://developer.mozilla.org/en-US/docs/WebAssembly
[`wasm.h`]: https://github.com/webassembly/wasm-c-api

These are some high-level descriptions of where the current API is not
sufficient, but it's worth going into some more detail on each point as well.

### Multithreading Wasmtime

The current implementation of the `wasmtime` crate has the side effect that any
type connected to a `Store` (`Linker`, `Func`, `Global`, ...) implements neither
the `Send` nor `Sync` traits. This means that all wasm types cannot be safely
sent to other threads in the Rust type system. It is desirable to do this,
however, in situations such as a multithreaded server which executes WebAssembly
for each input request. In this situation a runtime, such as `tokio`, will
require that futures for each request are `Send` and will migrate the future
between worker threads to ensure it's executed promptly (via workstealing and
such). Any futures with `wasmtime` types in them, however, are not `Send`.

Currently [there is documentation][multidoc] about multithreading in Wasmtime
which indicates that it is indeed safe to move a `Store` between threads, but
it's not possible to do so safely in the Rust type system. You would have to
ensure that all types you put in a `Store` (e.g. closure functions via
`Func::wrap` or `Store::{get,set}` values) are also safe to send across threads.
Furthermore the futures returned from `call_async` or host functions would
*also* have to be safely sendable to other threads. Guaranteeing all of this is
quite difficult to do so at scale and places a lot of burden on users who are
otherwise using Rust for its safety guarantees and properties.

Put another way, multithreaded runtimes (notably where futures are `Send`) are
an important use case for Wasmtime. Actually getting the `wasmtime` crate to
work in this situation, however, requires too much `unsafe impl Send` to
reasonably expect `wasmtime` to be embedded in this fashion. One of the goals of
this RFC is to enable this use case in safe Rust.

Multithreading is not only important for the multithreaded Rust runtime case
where `rustc` is relied upon for safety. Embeddings of Wasmtime are also likely
to use multiple threads. All of the wording about "you can send a store if you
send everything" in the Rust API side of things does indeed apply to C as well,
but just as it's difficult for Rust users to guarantee this is done it's also
difficult for C users to ensure that this is done. A good example of this is a
[recent bug report in `wasmtime-go`][gobug] where a `Store` was accidentally
shared across goroutines (and ended up being shared across threads).

Effectively the currently restrictive wording and requirements about using
Wasmtime on multiple threads today is infectious in that it affects *all* usage
of Wasmtime in all locations, not just Rust.

### Mutable access to host state

In the `wasmtime` Rust API host-defined functions always work through `Fn`.
This indicates that they can be recursively called (therefore not `&mut`
access). Additionally, though, it's expected that host state is closed over in
each host-defined closure, meaning that you only get shared (`&T`) access to
host state.

This design pattern requires pervasive use of interior mutability to implement
host state. For example `wasi-common`'s `WasiCtx` makes liberal use of
`RefCell` and `Cell`. While this is functional today it is a design pattern that
is non-obvious and is quite unergonomic. Beginners to Rust rarely get introduced
to interior mutability until much later on their Rust journey, and even Rust
veterans can find widespread interior mutability to be cumbersome to work with.

Interior mutability is not only an ergonomics issue, though, but it can also be
a correctness issue. For example it's easy to accidentally hold a `BorrowMut`
across a recursive wasm function call, not realizing that this will panic until
a wasm module recursively calls back into the host.

The current multithreading story of Wasmtime makes matters even worse in this
regard as well from the Rust side of things. The usage of `RefCell` within
`WasiCtx`, for example makes it so the type is not `Sync`. Additionally the host
wrappers will wrap this up in an `Rc` making it not `Send`. This means that it's
extremely difficult to find an "isolated" location to have `unsafe impl Send` to
try to have something multithreaded be safe in Rust. There end up being so many
reasons that a type is `!Send` or `!Sync` that embedders of Wasmtime effectively
need to just unsafely assert the entire world is always sendable to other
threads, making it very easy to mistakenly introduce a data race.

### Memory management

In the the Wasmtime API today all objects are distinctly owned. For example a
`Func` (or `wasm_func_t`) is a separate object from a `Table` (or
`wasm_table_t`) and a `Store` (or `wasm_store_t`). For the C API specifically
it's unclear what the intended semantics for ownership are with these types
according to the upstream header (the upstream `wasm.h` has no accompanying
documentation). For other languages (which are expected to be safe) and C (which
inherits our design decisions for the Rust API) each of these objects is
distinctly owned and is safe to live beyond all other objects.

This design decision means that for users of the C API there's quite a lot of
resource management and resource juggling to do. This means that C examples and
such have quite a few calls to `wasm_*_delete` just to manage the lifetime of
all the handles running around. Safe langauge embeddings must also be careful to
correctly run the destructor at the apppropriate time for each object. Note
that, as mentioned above, the difficulty of this is compounded by the
multithreading design of Wasmtime today which requires all objects are deleted
on the same thread, which is not easy to arrange for in embeddings like Go.

Separate from just juggling resources themselves another consequence of having
distinctly owned objects is that it's not always clear when resources are
released. For example if you use `Func::new` (or `wasm_func_new_with_env`) and
then immediately drop the `Func` (or call `wasm_func_delete`) this might
surprisingly not actually call the destructor for the closure. No data is
actually deallocated until the `Store` (or `wasm_store_t`) is deallocated. This
still, however, can be surprising where if you drop a `Store` (or call
`wasm_store_delete`) it's still not guaranteed to deallocate the underlying data
because there may be lingering other references to `Func` or `wasm_func_t`, for
example, keeping it alive.

These concerns around memory management are inherent in managed languages like
JS. Given that our API was heavily inspired by the JS API it's no surprise that
we inherited these intrinsic properties!

### A new API

With all of this context in mind, the goal of this proposal is to solve all of
these issues at once. The intent is to design an API that is much easier to use
safely with multiple threads, doesn't require somewhat obscure Rust idioms to
work with on the host side, and greatly simplifies the memory management and
resource ownership story. These properties are expected to affect all embeddings
of Wasmtime, regardless of language.

It's important to also note that the changes proposed here are not intended to
affect the implementation of Wasmtime itself. It's expected that very little
of the internals `wasmtime` will change. Cranelift and `wasmtime-*` private
crates are likely to remain the same after this API redesign. Namely this means
this proposal aims to not regress performance of any use case of Wasmtime today,
mostly just change how Wasmtime is expected to be used.

[multidoc]: https://github.com/bytecodealliance/wasmtime/blob/9637bc5a09ac00cbad2b890f455988613c542fb6/docs/examples-rust-multithreading.md
[gobug]: https://github.com/bytecodealliance/wasmtime-go/issues/60#issuecomment-823290616

# Proposal
[proposal]: #proposal

The first issue to tackle when redesigning the API of `wasmtime` is the
multithreading story. This influences so much of the rest of the API that any
solution for any other problem will probably look quite different depending on
how multithreading is approached!

## Uniquely owned `Store`

Today the `Store` type in `wasmtime` is a glorified piece of `Rc`-wrapped state.
Usage of `Rc` means that `Store` is neither `Send` nor `Sync`. Each object
connected to a `Store`, such as `Func`, then holds a strong `Rc` reference count
to the `Store`. This allows the `Store` to get dropped and the `Func` is still
usable. This design, however, is also the root of the lack of `Send` and `Sync`
on many `wasmtime` types. Naturally the `Rc` type is neither `Send` nor `Sync`,
but the bigger problem here is that having distinctly owned objects makes
transferring ownership to other threads significantly more difficult because
*all* objects have to be transferred. The crux of effectively this whole
proposal is reversing this, what if there was only one owned type in `wasmtime`?

The main proposal here is to make `Store` the only owned store-level type in
`wasmtime`. Internally `Store` will switch its `Rc` to `Box` which means it will
lose its `Clone` implementation as well. For types such as `Func` they will no
longer retain a reference to the `Store` itself internally. Instead the `Store`
must be passed as a sort of "context" argument to all methods on `Func`. By
disconnecting objects from a `Store` we immediately get a lot of benefits:

* If we want to safely send "everything" to one thread, that's quite easy, we
  just send a `Store`! Types like `Func` are easy to make `Send`/`Sync` if they
  don't retain any actual state of the function. `Func`, for example, is only
  functional if you pass in the right `Store`.

* Resource ownership and resource lifetimes are much easier in the C API. The
  only owned object is a `wasm_store_t`! Types like `wasm_func_t` will no longer
  need `wasm_func_delete` called on them since they won't own any resources.

* Multithreading now entirely hinges on `Store`. This is true for other
  embeddings as well. For example `wasmtime-go` would simply document "the
  `Store` type is not threadsafe", but everything else would indeed be
  threadsafe! Similarly for Rust, all that needs to be guaranteed is that
  `Store` is both `Send` and `Sync`. If this is achieved then multithreading in
  Rust is also safe as well (since you simply just move a `Store` to another
  thread via `Send`).

Currently in Wasmtime almost all methods also take `&self` instead of `&mut
self`. The reason for this is that everything is effectively behind an `Rc`
which means `&mut` wouldn't give any benefits (you can't do much with `&mut
Rc<T>` you couldn't do with `&Rc<T>`). With a uniquely-owned store, however, we
can start using Rust's `&mut` guarantees liberally. For example anything which
might mutate the internal state of a store (e.g. calling a wasm function), will
take `&mut self` instead of `&self`. This immediately provides the guarantee in
Rust that you can't invoke wasm concurrently within a `Store` (as is intended)
simply via the nature of Rust's type system. As we'll see later, too, using
`&mut self` for methods can enable `&mut` access to data in more places as well.
In any case there's still more multithreading bits to figure out first.

## `Store: Send + Sync`

If `Store` is uniquely owned then it's still not automatically thread-safe, we
need to make sure that all of its internals are indeed thread safe. To do this
we need to consider all objects which can be stored inside of a `Store`. Since
`Store` needs to be threadsafe, this means that all of these will also need to
be threadsafe.

The first items typically stored in a store are "host functions". These are the
`F` closures passed to `Func::wrap` or similar constructors. Currently these
types are all required to be `'static`, but for `Store` to be threadsafe then
these closures *also* need `Send` and `Sync` bounds. Due to the recursive nature
of WebAssembly these types will still need to be `Fn` as well. This seems quite
bad, though, because if all we get is `&T` and it needs to be `Send` and `Sync`,
then requiring mutability implies the need for locks and such!

A goal of this API redesign is to not incur the cost of synchronization where
not actually necessary (e.g. we're still not going to allow concurrent execution
of WebAssembly within a `Store`, so there shouldn't be any inherent need for
synchronization in host functions). To solve this issue the second inspiration,
after a uniquely owned `Store`, is to actually have `Store<T>`.

The idea here is that all host-state associated with a `Store` will be stored in
a `T` inside of the store itself. A `Store<T>` can be thought of a glorified
`Box<T>` in this regard. Using a type parameter here has a lot of nice benefits:

* If calling WebAssembly requires `&mut Store<T>`, we can propoagate that
  mutable borrow of the entire `Store` to invocations of host functions. This
  means that host functions can be given access to `&mut T`, removing the need
  for interior mutability.

* Using a type parameter `T` instead of some sort of trait object makes `Store`
  flexible for embeddings that *don't* want thread-safety. This means that
  `Store<T>` is conditionally `Send + Sync` depending on `T`. This way we're not
  forcing all users of Wasmtime to be `Send + Sync`. Instead users who want it
  can do it, otherwise embeddings are free to continue using `Rc` for host state
  as much as they'd like.

* Having a "shared" `T` amongst all host call implementations should make it
  much easier for embeddings to share data, in Rust, between different host
  functions. Currently this requires some form of `Rc<RefCell<T>>` to be cloned
  and closed over by each `Func`, but now state can all be stored in `T` itself.
  It's expected that after this change most closures passed to `Func::wrap` will
  actually be zero-sized closures.

The only other major item stored in a `Store` is values within an `ExternRef`.
At this time this RFC proposed changing `ExternRef` to use atomic reference
counting and require that all data inserted into it is `Send` and `Sync`. This
is the only behavior change proposed by this RFC, but is required to get
Rust-level safety guarantees about a `Store`.

It may be possible, however, to improve this situation. For example it might be
possible for `ExternRef` to have two reference counts where one is frobbed
during wasm execution non-atomically and the other is managed atomically on the
host. Exactly how this would work or exactly how this would be implemented is
not certain. It's predicted, though, that this change for `ExternRef` to be more
restrictive (requiring `Send` and `Sync`) as well as a bit slower (atomic
reference counting) is unlikely to have too much impact on embeddings today in
Wasmtime (assuming relatively little use of the reference-types proposal). If
this is the case then it's expected that the story around `ExternRef` can be
improved as the need arises over time.

## `Store` state in host functions

Today host functions in wasmtime can optionally take a `Caller<'_>` which
indicates the context of the calling instance. This is used for WASI to load the
caller's memory to figure out where pointers are relative to. The `Caller` type
also gives access to `&Store` too. Mentioned above this `Caller` type will now
also provide mutable access to the `T` inside of the `Store<T>`.

First this means that signatures that take `Caller<'_>` will now need to take
`Caller<'_, T>`. We might naively then provide access to `&mut Store<T>`, but
this would not actually be sound because then you could do something like
`*store = Store::default()` which would deallocate all wasm state. To handle
this there will instead be a `StoreContext` (name bike-sheddable of course) type
through which you access store methods:

```rust
impl<T> Caller<'_, T> {
    fn context_mut(&mut self) -> StoreContext<'_, T>;
}
```

A `StoreContext` can be thought of as `struct StoreContext<'a, T>(&'a mut Store<T>)`
but it won't be implemented precisely as this internally. Conceptually though
ownership of a `StoreContext` provides access to the `T` within the `Store`
itself. This all means that host state in functions will be accessed through the
`Caller` parameter's `Store`.

Astute readers at this point may be wondering what's going to happen to [shared
host functions][enginefunc]. Shared host functions are defined via
`Config::define_host_func` (and other related methods). Host functions, however,
are now required to have a `T` type parameter connecting them to the `Store<T>`
they can get hooked up to. Because `Config` does not have a `T` type parameter,
this will require an adjustment to how shared engine-level host functions are
created.

The gory details are below, but the general gist is that since all closures from
the host are required to be `Send + Sync + 'static` there's generally not a ton
of difference in "normal" host functions today and "engine" host functions. To
cover this difference the `Linker` type will now work at the `Engine` level
instead of the `Store` level. This allows creation of a `Linker` which can be
used in any `Store`, and internally it'll implement host functions in a similar
manner to engine-level functions today.

## Is this safe?

A very reasonable question to ask at this point is "wait is this all actually
safe?" When designing this proposal I was personally caught off guard about how
a `Caller` can't give access to `&mut Store<T>`, it wasn't immediately obvious
to me as to why from a safety perspective. If this is true, then why is giving
access to a `StoreContext` safe?!

This proposal claims (tentatively) that this API design indeed can be made safe.
To talk about some concrete Rust code, the envisioned implementation of `Store`
looks like:

```rust
struct Store<T>(Box<StoreInner<T>>);

struct StoreInner<T> {
    data: T,
    // OTHER STUFF
}

struct StoreContext<'a, T>(&'a mut StoreInner<T>);
```

When `StoreContext` is used to make calls into WebAssembly, what really ends up
happening is that `// OTHER STUFF` includes items like `InstanceHandle` values
and the invocation will happen through one of these pointers. During the call
we'd consider `InstanceHandle` values (and anything else used by WebAssembly) as
effectively borrowed.

If, however, a full `StoreContext` is given to callers when we call back into
the host, this is somewhat of a lie. The caller does not actually have an
exclusive borrow on the entire contents of the `StoreInner<T>` that it in theory
has access to. Frames below on the stack (wasm) will borrow data from the
`InstanceHandle` structures (and other miscellaneous things in `Store`).

In a sense, though, this can be roughly thought of as "what if `// OTHER STUFF`
was something like `Rc<OtherStuff>`". In a world like this `&mut StoreInner<T>`
does not imply exclusive access to `OtherStuff`. This is roughly the ownership
model that Wasmtime will want to use where `&mut StoreInner<T>` will imply
exclusive ownership and access to some fields in `StoreInner<T>` but not others
since it could be borrowed by other wasm frames on the stack.

The `Rc<T>` analogy breaks down though once mutability is considered. For
example creating an instance will create an `InstanceHandle` which means we'll
want to push it onto a `Vec<InstanceHandle>`. Currently this is behind a
`RefCell`, but there's no need for that with `&mut StoreInner<T>`. This means
that the ownership model here is more where if `&mut StoreInner<T>` is acquired
then mutation and access to state is allowed, but only **appending** mutation in
the sense that you can't invalidate any pointers held by WebAssembly. The
thinking here is that WebAssembly will have a bunch of pointers it assumes are
valid at some point in time, and then while the contents of those pointers can
change they're only ever accessed when mutable exclusive ownership is had by the
caller.

For example it would be invalid for `Func::call` to operate on `&StoreInner<T>`
because wasm usage of pointers requires mutable access, but that's not present
at that time. This means that the design of the `wasmtime` crate's API needs to
be quite careful to ensure that `&StoreInner<T>` and `&mut StoreInner<T>` is
chosen correctly for all APIs. It's hoped that the internals of `wasmtime` can
be structured in such a way that choosing incorrectly is very difficult, if even
possible, to do so.

All in all this boils down to the fact that `StoreContext` is passed to host
functions instead of `Store`. The presence of `Store` gives you the ability to
overwrite and invalidate existing pointers, which would be unsafe. Through
`StoreContext`, though, which we control via the API, it will not be possible to
invalidate existing pointers.

That being said `&mut StoreInner<T>` still does indeed imply mutable access to
owned interior members. For example you can create instances at any time, we
could trigger GC, etc. While pointers into `StoreContext` are held elsewhere
they're never used unless wasm is being called, for example, which required
mutable access to `StoreInner<T>` to get started.

Of course sanity checks on this aspect of the API are always greatly appreciated
from others!

[enginefunc]: https://github.com/bytecodealliance/rfcs/blob/main/accepted/shared-host-functions.md

## The new `wasmtime` Rust crate

With all the above context in mind, this is the proposed new API for the
`wasmtime` Rust crate.

### `Config` and `Engine`

```rust
impl Config {
    fn new() -> Config;

    // ... all existing builder methods ...

    // host function methods removed, added to `Linker` below.
}

impl Engine {
    // unchanged from current
}
```

### `Store`

The `Store` type is similar to as it exists today except that it uses a `Box`
internally instead of a `Rc`.

```rust
pub struct Store<T> {
    inner: Box<InnerStore<T>>,
}

struct InnerStore<T> {
    data: T,
    // ...
}

impl<T> Store<T> {
    fn new(engine: &Engine, data: T) -> Store<T>;

    fn context(&self) -> StoreContext<'_, T>;
    fn context_mut(&mut self) -> StoreContextMut<'_, T>;

    // ...
}

impl<T: Default> Default for Store<T> {}
```

Unlike today, however, a `Store` has no inherent methods. Instead you need to
acquire a "context" of some form to operate on the store ([alluded to
above](#store-state-in-host-functions)) without allowing destruction of the
store by overwriting it. These context are defined next.

### Context types and Traits

Two context types are provided which represent shared and mutable access to a
store internals. Most of the time users will operate with a `StoreContextMut`,
but not necessarily explicitly.

```rust
pub struct StoreContext<'a, T>(&'a InnerStore<T>);
pub struct StoreContextMut<'a, T>(&'a mut InnerStore<T>);

impl<T> StoreContext<'_, T> {
    fn get(&self) -> &T;
    fn interrupt_handle(&self) -> Result<InterruptHandle>;
    fn engine(&self) -> &Engine;
    fn get_host_func(&self, module: &str, name: &str) -> Option<Func>;
    fn fuel_consumed(&self) -> Option<u64>;
}

impl<T> StoreContextMut<'_, T> {
    fn get(&self) -> &T;
    fn get_mut(&mut self) -> &mut T;
    fn gc(&mut self);
    fn add_fuel(&mut self, fuel: u64) -> u64;
    fn out_of_fuel_trap(&mut self);
    fn out_of_fuel_async_yield(&mut self, injections: u32, amt: u64);
}
```

Note that for convenience all APIs on `StoreContext` will also be duplicated
onto `StoreContextMut`. Similarly all APIs on both context types will be
duplicated onto `Store` itself. While a bit of a bummer for us maintaining
`wasmtime` it should make it nicer for users to be able to work with whatever
they have. Documentation for these functions will primarily live on `Store` and
the methods on `StoreContext{,Mut}` will simply point to the `Store`
documentation.

All other existing methods in the `wasmtime` API are updated to take a "context"
argument. To assist with the ergonomics of this argument new traits are added as
well to the above two types:

```rust
trait AsContext {
    type Data;
    fn as_context(&self) -> StoreContext<'_, Self::Data>;
}

trait AsContextMut: AsContext {
    fn as_context_mut(&mut self) -> StoreContextMut<'_, Self::Data>;
}

// forward `&T => T`
impl<'a, T: AsContext + ?Sized> AsContext for &'a T {
    type Data = T::Data;
    fn as_context(&self) -> StoreContext<'_, T::Data> {
        T::as_context(self)
    }
}

// forward `&mut T => T`
impl<'a, T: AsContext + ?Sized> AsContext for &'a mut T {
    type Data = T::Data;
    fn as_context(&self) -> StoreContext<'_, T::Data> {
        T::as_context(self)
    }
}

// forward `&mut T => T`
impl<'a, T: AsContextMut + ?Sized> AsContextMut for &'a mut T {
    fn as_context_mut(&mut self) -> StoreContextMut<'_, T::Data> {
        T::as_context_mut(self)
    }
}

// Implementations for store-related and context types
impl<T> AsContext for Store<T> {
    type Data = T;
    fn as_context(&self) -> StoreContext<'_, T> { self.context() }
}
impl<T> AsContextMut for Store<T> {
    fn as_context_mut(&mut self) -> StoreContextMut<'_, T> { self.context_mut() }
}
impl<T> AsContext for StoreContext<'_, T> {
    type Data = T;
    fn as_context(&self) -> StoreContext<'_, T> { self.as_ref() }
}
impl<T> AsContext for StoreContextMut<'_, T> {
    type Data = T;
    fn as_context(&self) -> StoreContext<'_, T> { self.as_ref() }
}
impl<T> AsContextMut for StoreContextMut<'_, T> {
    fn as_context_mut(&mut self) -> StoreContextMut<'_, T> { self.as_mut() }
}
```

with these trait definitions you'll see parameters of one of two form in APIs
below:

1. `cx: impl AsContext` - this is for APIs which are safe to be called
   concurrently from many threads. For example learning about type information
   or something like `Global::get`.
2. `cx: impl AsContextMut` - this is the more common case for APIs which require
   exclusive access to call (therefore can't be called concurrently). This is
   used for cases which could mutate the store such as `Func::new` or
   `Func::call`.

The goal of these traits and these implementations is to make it as ergonomic as
we can to pass whatever kind of context you have into a function requiring a
context and it'll work.

### `Func`

Using the context types defined above the `Func` API will look similar to what
it is today except that methods now take a context parameter. There are two
subtle but impactful differences in this API worth calling out:

* All closures used to create a `Func` are required to be `Send + Sync`. This is
  because the types are stored in `Store` and we want `Store` to be `Send +
  Sync`. You still only get `Fn`, though, instead of `FnMut`. Mutability of host
  state is expected to go through the `T` on the `Store`, not the closed-over
  state of the function.

* Creation of an async host function requires the future returned to be `Send`,
  unconditionally. Unfortunately this isn't something that can be made
  parametric, it needs to be decided statically by the `wasmtime` API. At this
  time though it's seen as the most reasonable default since it matches the
  expected usage of `wasmtime` in async servers (where most futures are `Send`)
  and `Send` futures can always be created from non-`Send` ones via means like
  `spawn_local` or similar.

It's expected that most closures, as a result of the `Send+Sync+Fn()`
restriction, will be zero-sized and not actually close over any state. State
is expected to generally be accessed through the `Caller<'_, T>` passed in
instead.

```rust
struct Func(/* ... */);

impl Clone for Func {}
impl Copy for Func {}

impl Func {
    // constructors
    fn new<T>(
        cx: impl AsContextMut<Data = T>,
        ty: FuncType,
        f: impl Fn(Caller<'_, T>, &[Val], &mut [Val]) -> Result<(), Trap> + Send + Sync + 'static,
    ) -> Func;
    fn new_async<F, T>(cx: impl AsContextMut<Data = T>, ty: FuncType, f: F) -> Func
    where
        F: for<'a> Fn(
                Caller<'a, T>,
                &'a [Val],
                &'a mut [Val],
            ) -> Box<dyn Future<Output = Result<(), Trap> + Send> + 'a>
            + Send
            + Sync
            + 'static;
    fn wrap<T>(cx: impl AsContextMut<Data = T>, func: impl IntoFunc<T>) -> Func;
    fn wrapN_async(cx: impl AsContextMut, func: F /* ... */) -> Func;

    fn ty(&self, cx: impl AsContext) -> FuncType;
    fn typed<P, R>(&self, cx: impl AsContext) -> Result<TypedFunc<P, R>>;
    fn call(&self, cx: impl AsContextMut, args: &[Val]) -> Result<Box<[Val]>>;
    async fn call_async(&self, cx: impl AsContextMut, args: &[Val]) -> Result<Box<[Val]>>;
}

struct TypedFunc<P, R>(/* ... */);

impl<P, R> Clone for TypedFunc<P, R> {}
impl<P, R> Copy for TypedFunc<P, R> {}

impl<P, R> TypedFunc<P, R> {
    unsafe fn new_unchecked(func: Func) -> TypedFunc<P, R>;
    fn func(&self) -> Func;
    fn call<P, R>(&self, cx: impl AsContextMut, p: P) -> Result<R>;
    async fn call_async<P, R>(&self, cx: impl AsContextMut, p: P) -> Result<R>;
}

pub trait IntoFunc: Send + Sync + 'static {
    // ...
}
```

### `Global`

```rust
struct Global(/* ... */);

impl Clone for Global {}
impl Copy for Global {}

impl Global {
    fn new(cx: impl AsContextMut, ty: GlobalType, val: Val) -> Result<Global>;
    fn ty(&self, cx: impl AsContext) -> GlobalType;
    fn get(&self, cx: impl AsContext) -> Val;
    fn set(&mut self, cx: impl AsContextMut, val: Val) -> Result<()>;
}
```

### `Table`

```rust
struct Table(/* ... */);

impl Clone for Table {}
impl Copy for Table {}

impl Table {
    fn new(store: impl AsContextMut, ty: TableType, init: Val) -> Result<Table>;
    fn ty(&self, cx: impl AsContext) -> TableType;
    fn get(&self, cx: impl AsContext, idx: u32) -> Option<Val>;
    fn set(&self, cx: impl AsContextMut, idx: u32, val: Val) -> Result<()>;
    fn grow(&self, cx: impl AsContextMut, delta: u32, val: Val) -> Result<u32>;
    fn copy(
        cx: impl AsContextMut,
        dst_table: Table,
        dst_index: u32,
        src_table: Table,
        src_index: u32,
        len: u32,
    ) -> Result<()>;
    fn fill(&self, cx: impl AsContextMut, dst: u32, val: Val, len: u32) -> Result<()>;
}
```

### `Memory`

```rust
struct Memory(/* ... */);

impl Clone for Memory {}
impl Copy for Memory {}

impl Memory {
    fn new(cx: impl AsContextMut, ty: MemoryType) -> Result<Memory>;
    fn ty(&self, cx: impl AsContext) -> MemoryType;
    fn read(
        &self,
        cx: impl AsContext,
        offset: usize,
        buffer: &mut [u8],
    ) -> Result<(), MemoryAccessError>;
    fn write(
        &mut self,
        cx: impl AsContextMut,
        offset: usize,
        buffer: &[u8],
    ) -> Result<(), MemoryAccessError>;
    fn data<'a, T>(&self, cx: StoreContext<'a, T>) -> &'a [u8]; // safe!
    fn data_mut<'a, T>(&mut self, cx: StoreContextMut<'a, T>) -> &'a mut [u8]; // safe!
}
```

### `Instance`

```rust
struct Instance(/* ... */);

impl Clone for Instance {}
impl Copy for Instance {}

impl Instance {
    fn new(cx: impl AsContextMut, module: &Module, args: &[Extern])
        -> Result<Instance>;
    async fn new_async(
        cx: impl AsContextMut,
        module: &Module,
        args: &[Extern],
    ) -> Result<Instance>;
    fn ty(&self, cx: impl AsContext) -> InstanceType;
    fn exports(&self, cx: impl AsContext) -> impl ExactSizeIterator<Item = Export>;
    fn get_export(&self, cx: impl AsContext, name: &str) -> Option<Extern>;
    fn get_func(&self, cx: impl AsContext, name: &str) -> Option<Func>;
    fn get_typed_func<P, R>(&self, cx: impl AsContext, name: &str) -> Result<TypedFunc<P, R>>;
    fn get_table(&self, cx: impl AsContext, name: &str) -> Option<Table>;
    fn get_memory(&self, cx: impl AsContext, name: &str) -> Option<Memory>;
    fn get_global(&self, cx: impl AsContext, name: &str) -> Option<Global>;
}
```

### `Caller`

```rust
impl<'a, T> Caller<'a, T> {
    fn context(&self) -> StoreContext<'_, T>;
    fn context_mut(&mut self) -> StoreContextMut<'_, T>;
}

impl<T> AsContext for Caller<'a, T> {}
impl<T> AsContextMut for Caller<'a, T> {}
```

### `Linker`

With the change that all host functions, even those created within a `Store`,
must be `Send + Sync` the difference between a normal `Func` and Wasmtime's
current "engine host function" (defined in [RFC 9]) becomes much smaller than it
is today. This proposal takes this opportunity to simplify the way that engine
host functions are exposed to the user by changing the `Linker` API as well.

Previously a `Linker` was connected to a `Store`, meaning that it's only usable
within the context of a singular `Store`. Instead of a store this proposal
connects it to an `Engine`, making a singular linker usable across multiple
stores.

The expected usage is that previous users of `Config::define_host_func` and
similar functions will instead create a `Linker` after an `Engine` is created
(and before any stores are created). A `Linker` can then have functions defined
into it (which are not only conveniences now but also have entirely different
implementations than `Func::new`). A `Linker` then takes `impl AsContextMut` for
instantiation and getters which follows the "pass the context in" pattern and
allows for loading a host function into a `Store` (if necessary).

Note that a new API is also provided, `sub_linker()`. This method allows
creating a "child" linker which inherits all of the previous values set by the
parent but also allows defining new values in the returned value for
instantiation. This is intended to model how `Linker` today in `wasmtime`
inherits configuration from `Config`, and now this will still be possible with
`sub_linker` (and work for other types of items as well).

Finally note that `Linker` has a type parameter `T` to connect the `T` used for
defining host functions with the `T` that's required for `Store<T>` handles
passed in, ensuring that host functions are operating on the same state they're
given from a `Store`.

```rust
impl<T> Linker<'_, T> {
    fn new(engine: &Engine) -> Linker<'static, T>;
    fn sub_linker(&self) -> Linker<'_, T>;

    // Defining names within this `Linker`. Note that this will rarely be used
    // outside of a store since there won't be any `Extern` values to pass in.
    fn define(&mut self, module: &str, name: &str, id: impl Into<Extern>) -> Result<()>;
    fn define_name(&mut self, name: &str, id: impl Into<Extern>) -> Result<()>;
    fn alias(/*...*/);

    // Today this is a convenience function, but this proposal is where this
    // changes to being the implementation of today's engine functions.
    fn func_new(&mut self, module: &str, name: &str, func: F) -> Result<()>
        where F: impl Fn(Caller<'_, T>, &[Val], &mut [Val]) -> Result<(), Trap> + Send + Sync + 'static;
    fn func_wrap<P, R>(&mut self, module: &str, name: &str, func: impl IntoFunc<T, P, R>) -> Result<()>;
    // ... async versions with `func_new_async` and `func_wrapN_async` here too ...

    fn instance(&mut self, store: impl AsContext<Data = T>, name: &str, instance: Instance) -> Result<()>;
    fn module(&mut self, store: impl AsContextMut<Data = T>, name: &str, module: &Module) -> Result<()>;

    fn instantiate(&mut self, store: impl AsContextMut<Data = T>, module: &Module) -> Result<Instance>;
    async fn instantiate_async(&mut self, store: impl AsContextMut<Data = T>, module: &Module) -> Result<Instance>;

    fn iter(&self, store: impl AsContextMut<Data = T>) -> impl Iterator<Item = (&str, &str, Extern)>;
    fn get(&self, store: impl AsContext<Data = T>, ty: &ImportType<'_>) -> Option<Extern>;
    fn get_by_name(&self, store: impl AsContextMut<Data = T>, module: &str, name: Option<&str>) -> impl Iterator<Item = Extern>;
    fn get_one_by_name(&self, store: impl AsContextMut<Data = T>, module: &str, name: Option<&str>) -> Result<Extern>;
    fn get_default(&self, store: impl AsContextMut<Data = T>, module: &str) -> Result<Func>;
}
```

The `func_*` methods do not require a store to be passed in. Data for the
functions created (the `InstanceHandle`) will be owned by the `Linker`. When a
host function enters a `Store` (via instantiation or `get` in one form or
another) then the `Store` would store a reference to the functions owned by the
`Linker` to preserve the lifetime of the `InstanceHandle` for the entirety of
the `Store`. It's expected that this insertion operation would insert something
akin to `Arc<Mutex<Vec<InstanceHandle>>>` where the `Store` only holds it to
keep it alive and the `Linker` could continue to modify it if it so desired.
This means that inserting host functions into a `Store` should still be constant
time.

All type signatures for all host functions will be registered into the `Engine`
globally, which requires mutex-based synchronization. For programs which create
a singular top-level `Linker`, though, this cost is paid only once.


[RFC 9]: https://github.com/bytecodealliance/rfcs/blob/main/accepted/shared-host-functions.md

### WASI changes

The `wasmtime-wasi` APIs will need to change slightly to account for all the new
updates above. When outlining these changes this also is a good point to talk
about expected application design with this new API.

Previously in `wasmtime` each `Fn` put into a `Store` through a `Func` would
close over any state it needed to operate on. WASI historically
(pre-engine-functions) would effectively store this as `Rc<RefCell<WasiCtx>>`
and then clone that between all the `Fn` instances. With this proposed API,
however, this would not work because `Fn` is required to be `Send + Sync`, and
`Rc<T>` is neither of these.

Instead of each function closing over its own state it's expected that instead
functions will not close over any state (and be zero-sized) and instead state is
stored within the `T` of `Store<T>`. This is accessible in functions through the
`Caller<'_, T>` parameter passed in.

A reasonable question to ask, then, would be how does WASI get its context from
`T`? The expected implementation will be through traits. More concretely WASI
could be defined within a store for any `T` that defines how to get access to
`WasiCtx` from `T` itself. For example:

```rust
// in the wasmtime-wasi crate
fn add_to_linker<T>(funcs: &mut Linker<'_, T>)
    where T: AsMut<WasiCtx>;
```

This means that WASI functions could be pre-defined in any `Linker<T>`
where `T` provides access to `WasiCtx`. The implementation for each WASI
function would receive a `StoreContextMut<'_, T>` which it could then call
`get_mut()` to get `&mut T`, and finally `as_mut()` to get `&mut WasiCtx`. From
there WASI functions can operate as they usually do today.

The existing `Wasi` structure and methods like `Wasi::new` and
`Wasi::set_context` would be removed because the `WasiCtx` state is now expected
to be managed through `T` rather than captured internally by each function.

### Examples

To get an idea of what this API looks like in action, here's some examples
translated from the main `wasmtime` repository's examples:

#### "Hello, world!"

```rust
fn main() -> Result<()> {
    let mut store: Store<()> = Store::default();
    let module = Module::from_file(store.engine(), "examples/hello.wat")?;
    let hello = Func::wrap(&mut store, || println!("hello!"));
    let instance = Instance::new(&mut store, &module, &[hello.into()])?;
    let run = instance.get_typed_func::<(), ()>(&store, "run")?;
    run.call(&mut store, ())?;
    Ok(())
}
```

#### WASI

```rust
fn main() -> Result<()> {
    let engine = Engine::default();
    let module = Module::from_file(&engine, "foo.wasm")?;

    // pre-define wasi functions in an engine-level data structure to share
    // between stores
    let mut linker = Linker::new(&engine);
    wasmtime_wasi::add_to_linker(&mut linker)?;

    // Create the per-store WASI context and then the store itself.
    let wasi = wasmtime_wasi::WasiCtxBuilder::new()
        .inherit_stdio()
        .inherit_args()?
        .build()?;
    let mut store = Store::new(&engine, wasi);

    // Create the linker and get the default function to call from the module
    linker.module(&mut store, "", &module)?;
    let func = linker.get_default(&mut store, "")?;

    // And then call it!
    let typed = func.typed::<(), ()>(&store)?;
    typed.call(&mut store, ())?;

    Ok(())
}
```

## The new `wasmtime` C API

Given the above Rust API, the next piece to consider is the impact this has on
the C API for Wasmtime. In shape this Rust API looks extremely different from
`wasm.h` which does not have any sort of context handled passed around anywhere.
Additionally there's a bit of a disconnect where Wasmtime's implementation of
`wasm.h` involves objects like `wasm_func_t` being distinctly owned but the
context handles are only valid for an ephemeral period of time. Additionally the
C API as-is, with Wasmtime's interpretation of distinctly owned objects, has the
drawbacks mentioned in the [motivation] section around thread-safety and memory
management.

This RFC proposes the following course of action to cover these issues.

1. Continue to support `wasm.h` with a "compatibility" implementation.
2. Update `wasmtime.h` to more closely reflect the Rust API for all
   `Store`-related objects.

For users who want compatibility with purely `wasm.h` this will continue to be
an option, at the cost of some performance and some overhead. Object allocations
for types like `wasm_func_t` will be required and reference counting will need
to be layered on `wasm_store_t` under the hood. For users binding Wasmtime
specifically, though, the `wasmtime.h` API will serve as a no-overhead binding
relative to the Rust API, avoiding need for compatibility shims or extra
allocations.

## Support for `wasm.h`

As mentioned above Wasmtime will maintain support for the proposed upstream
`wasm.h`. This will be done effectively through a polyfill where all types will
look roughly like:

```c
#[repr(C)]
pub struct wasm_func_t {
    store: Arc<UnsafeCell<wasmtime::Store<()>>>,
    handle: UnsafeCell<StoreContextMut<'static, ()>>,
    func: wasmtime::Func,
}
```

In essence each `Store`-connected object will bundle a strong reference to the
`Store` and an interior pointer which is the handle to the store (which will
implement `AsContext` and such). The precise details here may not match this
since there may be some bits and pieces to take care of from a Rust soundness
perspective, but this is the rough shape.

This polyfill introduces overhead (e.g. `Arc`) and additionally doesn't expose
features of the Rust API (e.g. the `T` in `Store<T>`). This new functionality
will be exposed through the `wasmtime.h` header.

## Updates to `wasmtime.h`

This RFC proposes updating the `wasmtime.h` to more closely reflect the Rust
API where a context is explicitly passed around, and types like `Func` do not
have destructors. These changes also achieve the goals stated in the
[motivation] around thread safety where the only owned object for a store is
`wasmtime_store_t`. This will enable fixing issues in languages like Go, for
example, which may run destructors on separate threads and currently is
coordinated to send raw pointers to the same thread to get dropped.

The `wasmtime.h` header will still require functionality from `wasm.h` to work.
For example all type-related information (e.g. `wasm_functype_t`) will be used
as-is from `wasm.h` for learning about the types of items. The intention here is
that we'd like to use `wasm.h` where it works, but for parts where it doesn't
work we'll provide extensions (as-is the case today).

The `wasm_config_t` and `wasm_engine_t` types will be used from the `wasm.h`
header. The `wasmtime.h` header will provide extensions to these types but the
type definitions will remain in `wasm.h`. Everything from the `Store` and below,
however, will receive a parallel implementation in the `wasmtime.h` header.

### `wasmtime_store_t`

```c
typedef struct wasmtime_store wasmtime_store_t;
typedef struct wasmtime_context wasmtime_context_t;
wasmtime_store_t *wasmtime_store_new(
  wasm_engine_t *engine,
  void *data,
  void (*finalize)(void*)
);
wasmtime_context_t *wasmtime_store_context(wasmtime_store_t *store);
void *wasmtime_context_data(wasmtime_context_t *context);
void wasmtime_store_delete(wasmtime_store_t *store);
```

The constructor for a store takes arbitrary user-defined `data` along with an
optional finalizer for that data. This data is then retrieved from a
`wasmtime_context_t`.

The implementation of `wasmtime_store_context` is to call `context_mut` and then
return the underlying `&mut InnerStore<T>` as a pointer to the caller. This
yields the underlying store pointer to the outside world, and means
that a context pointer is unique to a store, no matter where it came from. The
documentation for this function will indicate that the `wasmtime_context_t`
pointer is strictly tied to the `wasmtime_store_t` pointer. It cannot be shared
across threads and typically should be stored adjacent to the store itself.

Operations on all objects in the rest of the `wasmtime.h` C API will operate on
`wasmtime_context_t` pointers.

### `wasmtime_val_t`

First up are the changes to the values going in/out of wasm.

```c
typedef struct wasmtime_val {
    wasmtime_valkind_t kind;
    wasmtime_valunion_t of;
} wasmtime_val_t;

typedef union wasmtime_valunion {
    int32_t i32;
    int64_t i64;
    float32_t f32;
    float64_t f64;
    wasmtime_func_t func;
    wasmtime_externref_t *externref;
    wasmtime_v128 v128;
} wasmtime_valunion_t;

#define WASMTIME_FUNC_NULL ((wasmtime_func_t) -1)

typedef uint8_t wasmtime_v128[16];

// externref
typedef struct wasmtime_externref wasmtime_externref_t;
wasmtime_externref_t *wasmtime_externref_new(void *data, void (*finalize)(void*));
wasmtime_externref_t *wasmtime_externref_clone(const wasmtime_externref_t *val);
void wasmtime_externref_delete(wasmtime_externref_t *val);
```

This representation is different from `wasm.h` in how it represents `funcref`
and `externref`. Additionally it has support for `v128` values. The
`wasmtime_func_t` type is defined below and null `funcref` values will be
represented with the `WASMTIME_FUNC_NULL` sentinel. The `externref`
representation supports null `externref` values with null pointers. Finally the
`v128` field is provided to represent `v128` values.

Unlike `wasm_val_t` a value of tyep `wasmtime_val_t` never owns its contents.
Consequently no destructor is provided. Usage of `wasmtime_val_t` in the below
APIs define the ownership semantics.

### `wasm_trap_t`

Traps will largely be used as `wasm_trap_t` from `wasm.h`, but there will be an
extension that does not require a `wasm_store_t` to create one:

```c
wasm_trap_t *wasmtime_trap_new(const wasm_message_t *message);
```

### `wasmtime_func_t`

This is the new API for functions:

```c
typedef uint64_t wasmtime_func_t;

typedef wasmtime_trap_t *(*wasmtime_func_callback_t)(
  void *env,
  wasmtime_caller_t *caller,
  wasmtime_val_t *args,
  size_t argc,
  wasmtime_val_t *results,
  size_t resultc
);

wasmtime_func_t wasmtime_func_new(
  wasmtime_context_t *context,
  const wasmtime_functype_t *ty,
  wasmtime_func_callback_t callback
  void *env,
  void (*finalizer)(void*)
);

wasm_functype_t *wasmtime_func_type(wasmtime_context_t *context, wasmtime_func_t);
wasmtime_error_t *wasmtime_func_call(
  wasmtime_context_t *context,
  wasmtime_func_t func,
  wasmtime_val_t *args,
  size_t argc,
  wasmtime_val_t *results,
  size_t resultc,
  wasmtime_trap_t **trap
);
```

Here only one constructor is provided (which takes caller-specified data and a
destructor, but the destructor can be `NULL`). For `wasmtime_func_call` both the
arguments and the results are considered owned by the caller. This is only
applicable with `externref` values flowing in and out, but the caller is
responsible for freeing the arguments eventually and also freeing any
`externref` results.

Note that no destructor is provided as it is not needed. Also note that
`wasmtime_func_t` is an "index-like" type where mis-usage will abort the
program. It is considered an embedder error to incorrectly use `wasmtime_func_t`
types rather than a normal program error, hence the process abort.

### `wasmtime_caller_t`

A `wasmtime_caller_t` provides access to the `wasmtime_context_t` under the hood
which allows recursive calls and recursive operations on objects within a store.

```c
typedef struct wasmtime_caller wasmtime_caller_t;

wasmtime_context_t *wasmtime_caller_context(wasmtime_caller_t *caller);
wasmtime_extern_t wasmtime_caller_export_get(
  wasmtime_caller_t *caller, const
  wasm_name_t *name
);
```

Note that this object is always only temporarily owned on the stack so no
destructor is necessary. Also note that from a caller's context you can
re-acquire the store-level data.

### `wasmtime_global_t`

```c
typedef uint64_t wasmtime_global_t;

wasmtime_global_t wasmtime_global_new(
  wasmtime_context_t *context,
  const wasm_globaltype_t *ty,
  const wasmtime_val_t *init
);
wasm_globaltype_t *wasmtime_global_type(
  wasmtime_context_t *context,
  wasmtime_global_t global
);
void wasmtime_global_get(
  wasmtime_context_t *context,
  wasmtime_global_t global,
  wasmtime_val_t *val
);
wasmtime_error_t *wasmtime_global_get(
  wasmtime_context_t *context,
  wasmtime_global_t global,
  const wasmtime_val_t *val
);
```

Like with functions the get/set APIs leave ownership of the `wasmtime_val_t`
with the caller for the `externref` variant.

### `wasmtime_table_t`

Unlike the `wasm.h` C API that has a unified `wasm_ref_t` there are duplicate
Wasmtime APIs for interacting with tables for either `funcref` or `externref`
types. It's expected that this may need an update or more APIs in the future if
more reference types are implemented.

Note that null `externref` values are represented with `NULL` and null `funcref`
values are represented with `WASMTIME_FUNC_NULL`.

```c
typedef uint64_t wasmtime_table_t;

wasmtime_table_t wasmtime_table_funcref_new(
  wasmtime_context_t *context,
  const wasm_limits_t *limits,
  wasmtime_func_t init
);
wasmtime_table_t wasmtime_table_externref_new(
  wasmtime_context_t *context,
  const wasm_limits_t *limits,
  const wasmtime_externref_t *init
);
wasm_tabletype_t *wasmtime_table_type(
  wasmtime_context_t *context,
  wasmtime_table_t table
);
uint32_t wasmtime_table_size(
  wasmtime_context_t *context,
  wasmtime_table_t table
);

wasmtime_error_t *wasmtime_table_funcref_get(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t index,
  wasmtime_func_t *val
);
wasmtime_error_t *wasmtime_table_externref_get(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t index,
  wasmtime_externref_t **val
);

wasmtime_error_t *wasmtime_table_funcref_set(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t index,
  wasmtime_func_t val
);
wasmtime_error_t *wasmtime_table_externref_set(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t index,
  const wasmtime_externref_t *val
);

wasmtime_error_t *wasmtime_table_funcref_grow(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t amount,
  wasmtime_func_t init,
  uint32_t *old_size
);
wasmtime_error_t *wasmtime_table_externref_grow(
  wasmtime_context_t *context,
  wasmtime_table_t table,
  uint32_t index,
  const wasmtime_externref_t *val,
  uint32_t *old_size
);
```

Like with functions the get/set APIs leave ownership of the `wasmtime_val_t`
with the caller for the `externref` variant.

### `wasmtime_memory_t`

This object will mirror `wasm_memory_t` as-is in `wasm.h` except context
pointers will be passed in like above.

### `wasmtime_extern_t`

Currently `wasm_extern_t` is a pointer with conversions to all the various
types, but with `wasmtime_func_t` and friends being value types this means that
`wasmtime_extern_t` can be defined as a simple union:

```c
typedef struct wasmtime_extern {
  wasmtime_extern_kind_t kind;
  uint64_t item;
} wasmtime_extern_t;

enum wasmtime_extern_kind_t {
  WASMTIME_KIND_FUNC,
  // ...
};
```

The accessors and conversions for `wasmtime_extern_t` are not necessary as they
are with `wasm.h` because the struct fields can be interpreted and read
manually.

### Other Changes

There are a number of other bits and pieces not mentioned here but the above
changes should cover all the relatively novel modifications to the C API and
everything else is expected to follow suit as necessary (with the idioms
outlined above).

The thread safety of the C API will be documented as the store/context pointers
cannot be shared across threads unless otherwise documented, but they can always
be moved to other threads and operated on. Certain functions, such as
`wasmtime_func_type`, will be documented as threadsafe to match the Rust
equivalent when only shared borrows are required.

The only destructor for a store will be `wasmtime_store_delete` which is
guaranteed to delete the entire contents of the store since it's an owning
reference. This will also be documented as invalidating all `wasmtime_context_t`
pointers into the store.

## Impact on Wasmtime in other languages

Language embeddings are in general left to decide what API best suits them for
exposing Wasmtime. The current embeddings of Wasmtime (outside of Rust & C) are
Python, .NET, and Go. These are expected to all change in a similar fashion to
each other, but will not mirror the specifics of the above Rust/C API where a
store context is passed around to all functions.

Implementation-wise an important point to consider for these languages is what
it looks like to call wasm from a host function callback. Naively implemented a
host function callback could close over a `wasmtime_store_t`, but then that
`wasmtime_store_t` would also be kept alive by the language object closing over
it. This cycle between the native language and Rust heaps would not be
collect-able and could result in a memory leak. To solve this the each embedding
will reflect the ownership of the Rust API (where a `Store` is the "owner" of
everything) and objects like `Func` will be considered as having a "weak"
pointer to the `Store` internally. This means that consumers will need to
persist the `Store` somewhere to keep it alive, but otherwise `Func` doesn't
contain a strong reference to the store itself. This allows host functions to
close over `Func` values (or other objects) safely without resulting in a leak.

The `Caller` context provided to host functions will be able to be used as the
argument to any functions which require a store argument (such as instance
creation). This means that even operations that require the store will not
require closing over the `Store` itself.

Other than this, though, the .NET, Go, and Python APIs will all look very
similar. `Func` will be callable without a store context (since it will store it
internally), and host functions can close over non-`Store` objects without
leaking. Furthermore deallocation in managed languages will be much simpler
since only the `Store` will have a registered finalizer. This will remove extra
synchronization done today in Go and [fix a `wasmtime-dotnet`
issue](https://github.com/bytecodealliance/wasmtime-dotnet/issues/53). Finally
this will make the lifetime of data living within a `Store` much more explicit
since there's only one "owned" handle to a store.

The multithreading story for these languages, from a user perspective, will not
be all that different from today. If users want Wasmtime's `Store` to cross
threads then they'll have to ensure that other pointers come along with it. It's
expected that this isn't too much of a burden though since this is roughly what
the memory safety story around multithreading already looks like within each
language.

## Cost of Generics

One of the nice parts about the existing `wasmtime` crate is that very little in
the crate is generic. Apart from the `Func` constructors and `TypedFunc` APIs
the `libwasmtime.rlib` compilation artifact contains the machine code for almost
everything. This means that consumers compiling against the `wasmtime` crate can
generally enjoy speedy incremental compiles when they develop against `wasmtime`
since usage of `wasmtime` doesn't monomorphize a whole mess of code into their
crate.

With `Store<T>`, however, this means that significantly more code will be
monomorphized into consumer crates than before. This can mean that incremental
compile times against `wasmtime` could take much longer or otherwise just be
higher in general if multiple values of `T` are present in a program.

It's expected, though, that the extra compile time cost here is generally
negligible compared to other Rust compile costs and otherwise the benefits in
this RFC outweigh this cost.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

With API design various bits and pieces are endlessly bike-sheddable. Some
alternatives are listed inline above with proposed new APIs. This section
doesn't cover these sorts of alternatives but instead intends to cover
ideas resulting in a fundamentally different of the API of `wasmtime`.

### Don't change anything

The first and foremost alternative to the API proposed in this RFC is the one
that we have today. Wasmtime's `Store` is indeed functionally safe to send to
other threads. Additionally the C API is perfectly functional as-is and is
already working for various language embeddings. The downside of this approach
is that you can't leverage the Rust compiler to ensure you sent a `Store` safely
to another thread. Additionally the C API has the drawbacks mentioned in the
motivation section. The current API, however, can much more easily be used to
implement the `wasm.h` proposed API. Additionally one could subjectively argue
that the current API of `wasmtime` is more ergonomic because the "state" (a
`Store`) is implicitly owned and passed through with all objects rather than
needing to explicitly be passed through. Given the pervasive use of interior
mutability in today's Wasmtime API, however, the argument that the existing API
is more ergonomic is by no means a slam dunk argument.

### Retrofitting synchronization

An important alternative to consider, however, is what it might look like to
make the current API safe with respect to thread safety. Everything in the API
takes `&self` which means that some form of synchronization will be required,
aka a lock. This lock would need to be a `ReentrantMutex` of some form where the
same thread can acquire the mutex multiple times, and this could serve as a sort
of store-level-"GIL". Using a lock like this means that state in `Store` only
needs to be `Send`, not necessarily `Sync`, meaning that usage of `RefCell` and
`Cell` for closure state would still be permissible.

This alternative with locking, however, primarily breaks down when considering
the futures returned by `call_async`. There's two aspects to this which are
important to consider:

* First we would need to implement a custom form of reentrant mutex. The goal is
  for the futures returned by `call_async` to be `Send`, meaning they can be run
  on multiple threads over time. Upon entry into `call_async` we would need to
  acquire the reentrant mutex into `Store`. After acquisition when a future
  wants to be suspended we still want the mutex to be held, though. This mutex,
  when resumed on another thread though, has had ownership of the lock
  transferred to another thread. Typical `ReentrantMutex` RAII guards are not
  `Send`, so naturally Wasmtime would have to implement its own. Even if this
  were done, though, it's not 100% clear that this is even safe to do so. For
  example if a thread was calling some wasm, then it produced a purportedly
  `Send` future in a host call, there's nothing really stopping it from sending
  that future to another thread and then continuing to operate on the `Store` on
  the original thread (resulting in unsafe concurrent access).

* Second code generators such as `wiggle` would need to change their generated
  code to ensure that returned futures are `Send` without requiring locks
  everywhere. Today `wiggle`, for async users, would generate a trait that looks
  roughly akin to:

  ```rust
  trait HostFunctions {
      async fn the_function(&self);
  }
  ```

  The problem with this, though, is that `&self` is captured in the returned
  future from `the_function`. For the returned future to be `Send` that then
  requires `Self: Sync`, which then requires any mutable state in `Self` to have
  locks or other forms of synchronization. The lowest-runtime-overhead way to
  fix this is likely the usage of something similar to tokio's `TaskLocal`,
  which means the trait would instead look like:

  ```rust
  trait HostFunctions {
      async fn the_function(me: TaskLocal<Self>);
  }
  ```

  but this makes accessing state in `me` much less ergonomic. With this proposal
  the wiggle-generated trait would look like:

  ```rust
  trait HostFunctions<T> {
      async fn the_function(store: StoreContextMut<'_, T>);
  }
  ```

  where accessing state can simply be `store.get_mut()`.

Note that these limitations above are only related to expressivity and where
locks would be required (and some implementation requirements). This doesn't
even start to get into how every single wasm operations would require taking and
releasing a lock as well as accessing a thread-local to figure out what the
current thread is. With wasmtime's per-function-call overhead at around ~20ns
right now the addition of this overhead would likely at least double this
overhead (and we would have no recourse to improve it).

These limitations overall are what led to the conclusion that retrofitting
`Send` and `Sync` on the current API is somewhat of a dead end. It's unclear if
it can be feasibly implemented and the hoops that users of async would have to
go through to get `Send` futures are quite numerous.

# Open questions
[open-questions]: #open-questions

- Is it worth polyfilling the `wasm.h` API as proposed above? Or should support
  for `wasm.h` be dropped?

- If `ExternRef` moves to atomic reference counting and requires `T: Send +
  Sync`, is that feasible? Are there use cases today that rely on where this
  forces too much overhead? Along similar lines, are there possible designs for
  `ExternRef` which relaxes some of these restrictions?

  One perhaps saving grace is that hosts themselves are unlikely to frob the
  reference count that much in Rust. In Rust you'll often work with `&ExternRef`
  which then allows you to work with a borrow rather than updating the reference
  count every time it's passed around. Overall this means that updates to the
  reference count should be rare-ish in Rust (similar to how they're rare-ish
  with `Arc<T>`).
