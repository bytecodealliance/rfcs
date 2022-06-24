# Summary

[summary]: #summary

This RFC proposes adding native support to the `wasmtime` crate for imported
functions which are `async`. Additionally it provides APIs to call wasm
functions as an `async` function.

# Motivation
[motivation]: #motivation

WebAssembly today firmly lives in the "synchronous" world of computation.
Current stable WebAssembly does not have any support for stack-switching or any
standard form of handling asynchronous programs (e.g. something with an event
loop). There are eventual proposals for doing so, but they're expected to be
somewhat far out.

Rust, however, has an asynchronous ecosystem built around the `Future` trait
in the standard library. This means that if you have an asynchronous host
function you'd like to expose to WebAssembly, it's very difficult to do so today
with Wasmtime.

We have also seen a number of points of interest in Wasmtime supporting
something akin to stack switching, frequently to enable async functions with
wasmtime:

* The [`wasmtime-async`](https://crates.io/crates/wasmtime-async) crate.
* [Lucet](https://github.com/bytecodealliance/lucet) supports async
  imports/calls.
* [Requests to run lots of wasm on a
  server](https://github.com/bytecodealliance/wasmtime/issues/642)
* [Lunatic](https://github.com/lunatic-lang/lunatic) is an example project using
  Wasmtime which has stack switching under the hood.

A frequent use case of Wasmtime is to power server-side WebAssembly execution
where the server may be executing many WebAssembly modules simultaneously. It's
expected that many of these modules will be waiting on I/O or host
functionality, and having a thread-per-module will be too costly in terms of
synchronization and resources.

The goal of this RFC is to specifically target the `async` use case in Rust.
While the implementation will involve stack-switching and some communication
between the two stacks, it does not expose an API to users which allows
manually doing low-level operations like stack switching, yielding, and
resumption.

# Proposal
[proposal]: #proposal

This RFC proposes adding `async` support natively to the `wasmtime` crate
itself. While it's perhaps possible to build this externally (as shown by
`wasmtime-async`) it's a requested enough feature that adding core support
should increase its visibility and help it feel more well supported.

When adding support for the `wasmtime` crate, however, this proposal also aims
to not disturb the functionality that Wasmtime has today. Not all users of
Wasmtime need to handle `async` code and/or switch stacks. It should still be
possible, for example if necessary, to build Wasmtime without `async` support.
This effectively means that this proposal isn't changing existing APIs in
`wasmtime`, instead augmenting existing ones. Additionally, however, we want to
avoid "split the world" scenarios as much as we can. For example `Memory`
doesn't have much reason to be different in async and non-async worlds, and same
with `Module`.

### Changes to `Store`

With these goals in mind, the first change to Wasmtime is a new
constructor for `Store`:

```rust
impl Store {
    pub fn new_async(engine: &Engine) -> Store {
        // ...
    }
}
```

This `new_async` method sets a runtime flag within `Store` intending to indicate
that it is to be used with asynchronous methods defined below. The goal of this
method is to prevent mistakes, not to enable any sort of internal functionality
at this time. This'll be a bit more clear below as we cover the changes to
`Func`.

### Creating a `Func`

The first location where dealing with `async` functions on the host comes into
play when you're creating a `Func` from host-defined functionality. The existing
`Func::new` and `Func::wrap` constructors will continue to work in both async
and non-async `Store` objects. In additon to these constructors, however, new
constructors which take asynchronous functions will be added:

```rust
impl Func {
    pub fn new_async<'a, T, R>(
        store: &Store,
        ty: FuncType,
        state: T,
        func: fn(Caller<'a>, &'a T, &'a [Val], &'a mut [Val]) -> R,
    ) -> Func
    where
        R: Future<Output = Result<(), Trap>> + 'a,
    {
        // ...
    }

    pub fn wrap_async<T, P, R>(store: &Store, state: T, func: impl IntoFuncAsync<T, P, R>) -> Func {
        // ...
    }
}

impl<'a, T, A, Fut, R> IntoFuncAsync<T, (A,), R> for fn(&'a T, A) -> Fut
where
    Fut: Future<Output = R> + 'a,
    A: WasmTy + 'a,
    R: WasmRet, // includes `T`, (), and `Result<T, Trap>`
{
    // ...
}

impl<'a, T, A, Fut, R> IntoFuncAsync<T, (A,), R> for fn(Caller<'a>, &'a T, A) -> Fut
where
    Fut: Future<Output = R> + 'a,
    A: WasmTy + 'a,
    R: WasmRet, // includes `T`, (), and `Result<T, Trap>`
{
    // ...
}
```

These are intended to mirror the `new` and `wrap` functions, except they're
bound to functions returning futures instead. These signatures are a bit
wonky, however, and worth going into more detail of.

First, these functions will panic if the `Store` passed in is not an async
store. This is hoped to prevent a footgun where you hook up an asynchronous
function into a context that's expected to be called synchronously. There's no
real way for us to make this work so we want the programming error to get
reported ASAP.

Semantically what these instances of `Func` are expected to do is that when
they're invoked they'll be responsible for `poll`-ing the `Future` returned.
When the future is "pending" we'll stack-switch back to the original future that
called us (to be described later). When the future is ready we'll return the
value back to wasm. This way the wasm sees a "blocking" call to an imported
function, when in fact we just suspended the instance temporarily.

The weirdest part about these signatures, however, is the fact that you're
passing in an explicit state (`T`) and working with a function pointer (`fn`)
instead of some sort of closure (unlike `new` and `wrap` which use `Fn`). The
reason for this is effectively that [async fn in traits is
hard](http://smallcultfollowing.com/babysteps/blog/2019/10/26/async-fn-in-traits-are-hard/).
More concretely what we *want* to write is:

```rust
pub fn new_async(
    store: &Store,
    ty: FuncType,
    func: impl AsyncFn(Caller<'_>, &[Val], &[Val]) -> Result<(), Trap>,
) -> Func
{
    // ...
}
```

but the `AsyncFn` trait does not exist today, nor does Rust have the support to
make it exist. An alternative signature is perhaps:

```rust
pub fn new_async<'a, R>(
    store: &Store,
    ty: FuncType,
    func: impl Fn(Caller<'a>, &'a [Val], &'a [Val]) -> R,
) -> Func
where
    R: Future<Output = Result<(), Trap>> + 'a,
{
    // ...
}
```

but this does not allow the output future to borrow the captured environment of
the closure, which is expected to be a common operation. This exposes one of the
key weakneses of async-fn-in-traits which is that there's no way right now to
tie the lifetime of `&self` in the function call to the output future. While
perhaps solvable in the future, there's other problems to tackle as well, and
we'd like to ship something in the meantime!

Coming back to our original signature:

```rust
pub fn new_async<'a, T, R>(
    store: &Store,
    ty: FuncType,
    state: T,
    func: fn(&'a T, Caller<'a>, &'a [Val], &'a mut [Val]) -> R,
) -> Func
where
    R: Future<Output = Result<(), Trap>> + 'a,
{
    // ...
}
```

By using a function pointer we are able to connect the lifetime of the input
state and all the aguments to the output future, meaning it's allowed to borrow
any of its inputs, as you'd expect in `async` Rust. Example usage of this might
look like:

```rust
struct MyState;

impl MyState {
    async fn foo(&self) -> i32 {
        // ...
    }
}

let state: MyState = ..;
let func = Func::wrap_async(&store, state, |s| async move { s.foo().await });
```

This is still not as ergonomic as `Func::wrap`, but it's hoped that it's about
the closest that we can get for now.

### Calling a `Func`

After we've created a slew of `Func` instances backed by asynchronous functions,
then at some point we need to actually call wasm code! This is currently done
with `Func::call`, `Func::get`, and `Instance::new` today. Each of these will
get an async counterpart.

First, however, each entry point of calling a function will panic if it's done
from the wrong store. For example if `Func::call` is used on an async store,
then `Func::call` will panic. Similarly if you use `Func::get2` or
`Instance::new` on an async store the function will panic. Like above with
creating a `Func`, the goal is to signal a programmer error ASAP instead of
deferring it to "this panics only if called synchronously and happens to call an
async import".

Calling a function is expected to be a pretty straightfoward addition:

```rust
impl Func {
    pub async fn call_async(&self, params: &[Val]) -> Result<Box<[Val]>> {
        // ...
    }
}
```

and same with instantiation:

```rust
impl Instance {
    pub async fn new_async(store: &Store, module: &Module, imports: &[Extern]) -> Result<Instance> {
        // ...
    }
}
```

The `Func::get*` suite of methods are a bit more complicated but intended to
still be relatively straightforward:

```rust
impl Func {
    pub fn get1_async<A, R>(&self) -> Result<impl Fn(A) -> FuncCall<R>>
    where
        A: WasmTy,
        R: WasmTy,
    {
        // ...
    }
}

pub struct FuncCall<R> {
    // ...
}

impl Future for FuncCall<R> {
    type Output = Result<R, Trap>;
    // ...
}
```

These will all *panic* (not return an error) if called within a non-async store.

In all of these cases the futures returned, when polled, will execute
WebAssembly code. This is more fully described in the next section. The futures
is only pending if an import is called whose future winds up returning that it's
pending.

### Execution trace of async wasm

Here we'll discuss a (hopefully) complete picture of what executing WebAssembly
asynchronously looks like. Some details will be eschewed here and there but
it's intended that all the pieces above can be connected.

1. The first step is that the embedder will invoke some WebAssembly
   asynchronously. This is done via one of the three entry points of
   `Func::call_async`, `Func::get_async` (and then calling it), or
   `Instance::new_async`.

2. The entry point's future, let's call it future A, is eventually polled. As
   this is the first time A is polled, we need to set up a new native stack,
   let's call it stack 2 and our current stack stack 1, for execution that can
   be suspended. The future A will allocate stack 2 from the `Store` (probably
   through some sort of configurable trait) and then initialize stack 2 so,
   when started, it will call the appropriate wasm function. The exact details
   of the switch will be left to the implementation, but the idea is that some
   assembly code will be involved.

3. Some native trampoline code initially executes on stack 2 which does
   any setup necessary, and then jumps to the native wasm code, still executing
   on stack 2.

4. WebAssembly executes for awhile on stack 2, doing its thing and
   executing for a bit. Note that any synchronous host functions are no
   different in this scenario, they'll just execute to completion when called.

5. Eventually an imported function is called which was defined with an
   asynchronous Rust function. This first invokes the asynchronous function
   which returns a future, called future B. Using the polling context we had
   originally in step 2 when polling A, we poll the future B.

6. Future B is not ready. This triggers a transition back to stack 1. Future A's
   `poll` method receives informationg through a side-channel (probably in
   `Store` or something like that) about the status of what happened on stack 2.
   It sees that future B is not ready, so future A says it is not ready.

7. When the runtime deems appropriate, we are then polled again (still on stack
   1). Future A's `poll` method runs again and sees that this is the second time
   polling. It switches to stack 2 (stored internally) and resumes execution.

8. On stack 2 we re-poll future B, which was pinned to stack 2 and as a result
   hasn't moved. If B isn't ready we restart from 6, but let's say it's ready.
   This means we return from the native trampoline back to the wasm code.

9. Wasm code continues executing on stack 2. If it calls more asynchronous
   imports we resume from step 5. Otherwise let's say the wasm returns.

10. After the wasm returns we are returned to the original native trampoline
    on stack 2. We squirrel away the return values and then switch to stack 1.
    Back on stack 1 we see that execution finished. After deallocating the stack
    we then return that the future is done.

Note that this is all somewhat slow so the initial implementation probably won't
be too heavily optimized like the `Func::get*` specialized trampolines. It'll
likely use the `Func::call`-style trampolines to transmit a variable number of
arguments through the stack to avoid having lots of different entry points.

### Cancellation

In the Rust asynchronous world if you want to cancel a future you simply `drop`
it, effectively deallocating its resources. In the trace above we have three
distinct drop points. If we drop the future between steps 1/2, before we ever
polled, then no stack was ever allocated so it's easy to deallocate. Dropping
after step 10, after the future is done, is also easy because there is no stack.
Dropping between 6/7, however, is the interesting part.

If the future is dropped while wasm is suspended, then we need to clean up the
stack. To do this we need to unwind all frames on the stack and run destructors,
we can't simply just deallocate the stack to the original allocator. The current
thinking of how to do this is that we'll induce traps.

When (in the above example) future A is dropped between steps 6/7, the following
will happen:

* In the destructor of future A we'll resume execution on stack 2.

* Execution on stack 2 will receive the signal that it's been canceled. This
  will drop the future B and then return a trap from the host call, simulating
  that the imported function returned a trap.

* Execution of the wasm will proceed as usual if a trap otherwise happened.

* Eventually we'll reach the original native trampoline code on stack 2, which
  will switch back to stack 1 indicating that a trap happened.

* Back on stack 1, inside future A's destructor, we'll discard the result of the
  execution, a trap, and deallocate the stack now that there are no more frames
  on it.

The trickiest part about this is that if wasm/native code are interleaved on the
stack then native code can "catch" the "you've been cancelled" trap and resume
execution as usual, possibly taking a long time to return. This is, however,
also true of "you've been interrupted" traps which can also be caught. It's
expected that native code mostly just forwards traps along so this won't be much
of a problem (despite it being a possibility). If this issue is urgent,
however, we could consider adding a mechanism to `Store` which "poisons" wasm
execution or similar and prevents reentering any wasm module and always returns
a trap if wasm is called. This way while we're running A's destructor it will
prevent any further entry into wasm.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

One of the main restrictions of this proposal is that the only API surface area
added to `wasmtime` is a few `async`-aware functions. This intentionally leaves
out low-level API knobs for yielding/resumption/stack switching. The hope is
that almost all consumers will be able to get by with only using `async`.
Designing support for the highest priority use case also hopefully helps keeps
us flexible as future wasm proposals get fleshed out and may want to get added
to wasmtime as well. Ideally we wouldn't accidentally box ourselves into a
particular implementation which ends up being difficult to add a future wasm
proposal.

Another goal of this proposal was to integrate well into the `wasmtime` crate's
existing API, but it still unfortunately has a "split world" problem with
calling and creating functions. It's unknown whether this will make
wasmtime-as-a-library difficult to use if contexts aren't able to easily reason
about whether async should be used or not. It's hoped, however, that given the
purpose of Wasmtime that this is not a heavily shared dependency amongst many
crates but rather a component of a larger application which is able to make
these sorts of policy decisions.

# Open questions
[open-questions]: #open-questions

- All futures returned by `wasmtime` will not be `Send`. This is due to the fact
  that nothing relating to a `Store` is `Send`. How does this impact expected
  use cases of running `wasmtime` with asynchronous imports? Is this too
  restrictive? If too restrictive, how is this expected to be solved?
