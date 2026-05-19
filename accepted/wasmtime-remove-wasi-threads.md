# Summary

[summary]: #summary

Delete the implementation of the [wasi-threads] proposal from Wasmtime, namely
the `wasmtime-wasi-threads` and `wasi-common` crates, along with the `-Sthreads`
CLI flag. Uses cases that need any threading at all for portability will be
served in the near term with WASIp3 cooperative threads, and long-term support
for true multithreading will be best served through the
[shared-everything-threads] proposal

# Motivation
[motivation]: #motivation

This section intends to outline the various means and motivations for why
[wasi-threads] is being removed. This goes into spec-level motivation as well as
Wasmtime-level motivation as well.

## Limitations of wasi-threads

The [wasi-threads] proposal is a proposal for WASI which adds the ability to
spawn OS-native threads for workloads that require parallelism and using
multiple CPU cores. This proposal is, in its own words, now considered a "legacy
proposal" and notably is not compatible with the component model, nor is it
possible to make it eventually compatible with the component model. The
[wasi-threads] proposal fundamentally follows the path of multithreading in
browsers which is to model threads as WebAssembly instances. There are a number
of drawbacks to this model of threading, shared with the web, such as:

* Re-instantiating a core module requires re-supplying all imported functions
  whenever a new thread is spawned. This requires the host to decide whether to
  share state amongst all threads, give unique state to each thread, etc.

* Mutations to non-shared items in a WebAssembly module are not preserved across
  threads. This is desired behavior for WebAssembly `global`s to behave like
  thread-locals, but this means that mutations to the per-instance table is not
  shared across threads. This makes `dlopen`-style cases fundamentally
  incompatible with wasi-threads for example.

* The [wasi-threads] proposal is fundamentally incompatible with the component
  model due to component model intrinsics not having semantics across threads
  (e.g. they all specifically assume single-threaded semantics and execution).

* The [wasi-threads] proposal has fundamental open questions around topics such
  as thread lifecycle and TLS destructors which have no known answers at this
  time.

Overall [wasi-threads] is a minimal proposal to provide access to true parallel
execution to WebAssembly, but beyond that it has known fundamental limitations
which prevent it moving beyond the minimal proposal that it is now.

## Successor to wasi-threads

These limitations are not new or novel in any way, and this is one of the
motivations for the [shared-everything-threads] proposal.

The [shared-everything-threads] proposal is a core WebAssembly proposal which is
understood to be the effective successor of the [wasi-threads] proposal, in
spirit at least. Chiefly the [shared-everything-threads] proposal avoids the
instance-per-thread model of [wasi-threads] which ends up naturally solving
many of the issues above. For example modules aren't re-instantiated, so there's
no need to provide imports again. All items used across threads are marked as
`shared`, so for example table mutations (and `dlopen`) are supported.
Additionally there is a clear path forward to integrating
[shared-everything-threads] with the component model with its intrinsics and
such.

Current spec-level effort for multithreading in WebAssembly is primarily focused
on the [shared-everything-threads] proposal. The instance-per-thread model on
the web works as an "any means necessary" method of achieving parallelism, but
languages and toolchains have had difficulty integrating well with this model
given its severe limitations.

The downside of the [shared-everything-threads] proposal, however, is that it is
likely years away from being ready. Much of the current effort around it is
focused on shared GC types, for example, and shared functions, tables, and
globals are expected to come later. This means that waiting for
[shared-everything-threads] is not necessarily a viable alternative for
embedders which require true parallelism today.

## Shared memory in Wasmtime

Wasmtime has had an implementation of the [threads] proposal of WebAssembly for
some time, which [wasi-threads] is built on and the instance-per-thread model is
built on as well. Wasmtime's implementation, however, is [known to not be
sound](https://github.com/bytecodealliance/wasmtime/issues/4203). Wasmtime's
current implementation of shared memory additionally has known limitations such
as [not integrating with the ability to limit memory
growth](https://github.com/bytecodealliance/wasmtime/issues/4240) and [not
having the ability to customize thread blocking
behavior](https://github.com/bytecodealliance/wasmtime/issues/12809). Wasmtime
additionally has no built-in means to control thread lifetimes, such as shutting
down a store quickly and deleting all threads. Wasmtime also has no ability to
limit thread spawning built-in.

Wasmtime's implementation of core WebAssembly shared memory is currently gated
behind a CLI flag and `Config` feature. This reflects the immaturity of the
implementation relative to the rest of Wasmtime. As the fundamental primitive of
[wasi-threads], this effectively does not bode well for Wasmtime's quality of
implementation.

## Wasmtime and `wasi-common`

Wasmtime's original implementation of WASIp1 lives in a crate named
`wasi-common` inside of the Wasmtime repository. Prior to WASIp2 and the
component model this was Wasmtime's primary means of implementing WASIp1.
Nowadays, however, `wasi-common` is not used by default and instead the
`wasmtime-wasi` crate is used to implement WASI instead. The `wasmtime-wasi`
crate has a fundamentally different design than the `wasi-common` crate,
primarily motivated with integration with the component model. This includes
features such as `async` which WASIp1 does not support. The `wasi-common` crate
is only used by the `wasmtime` CLI when the `-Sthreads` flag is passed, which
enables [wasi-threads]. This is because the `wasmtime-wasi` crate is not
compatible with [wasi-threads] and thus `wasi-common` is the only means of
calling WASIp1 APIs when threads are in use.

Over time, however, `wasi-common` has received little-to-no maintenance since
`wasmtime-wasi` became the default. The `wasi-common` crate has remained a
vestige of Wasmtime's original implementation of WASIp1 and the means of
implementing WASIp1 when [wasi-threads] is enabled, but that's it. The
`wasi-common` crate has a growing set of rapidly aging dependencies which
require more and more effort to update and/or remove the dependency of.
Essentially `wasi-common` is becoming more and more of a maintenance burden over
time.

## WASIp3 and Coop threads

On the near-term horizon (likely before the end of 2026) the WASIp3
specification is expected to require cooperative threads in the component model.
This will provides the ability for guests to spawn threads and use
`pthread`-style APIs, for example. This support specifically requires
cooperative threading, however, which means that true CPU-level parallelism will
still not be possible.

Despite this, however, there are applications which primarily require threads to
be run on Wasm because removing threads is too onerous or too difficult. These
use cases are expected to be well-served by WASIp3 cooperative threading as it
will enable porting to WebAssembly with minimal modifications.

## Maintainer investment

No currently employed maintainer of Wasmtime has an interest in supporting and
maintaining the [wasi-threads] proposal or implementation within Wasmtime.
With maintenance costs increasing over time it's become unsustainable to
continue to keep this implementation.

Wasmtime's current maintainers do not have the resources to prioritize the
development of parallel execution of WebAssembly in Wasmtime. This includes
improving Wasmtime's implementation of shared memory, but also helping to
develop and advance the [shared-everything-threads] proposal. The reality of
today's situation is that there are not committed resources to improve this
situation, hence the proposal for removal.

## Removing wasi-threads and `wasi-common`

In the end the [wasi-threads] proposal is now considered legacy and receiving no
further development. All future development is expected to go behind
[shared-everything-threads] instead which will solve many of the fundamental
problems of [wasi-threads] and enable integration with the component model, for
example. While [shared-everything-threads] is not expected to be done for a
number of years, in the meantime the WASIp3 cooperative threading implementation
will suffice for applications that "just need threads" to be ported to
WebAssembly.

The `wasi-common` crate is a large, legacy crate in Wasmtime's codebase which
has a growing maintenance that is only increasing over time. There is no use for
`wasi-common` outside of [wasi-threads] additionally.

Overall the conclusion, and proposal of this RFC, is to relieve Wasmtime of
these burdens. Specifically the [wasi-threads] implementation and `wasi-common`
crate will be removed from Wasmtime. Users who currently rely on these
components will need take action, such as:

* Port to WASIp2 or WASIp3 without the use of threads.
* Port to WASIp3 cooperative threading when it is ready.
* Implement WASIp1 APIs themselves on top of the `wasmtime` crate, enable shared
  memory (despite known drawbacks in Wasmtime), and carry an implementation of
  [wasi-threads] themselves.
* Use a historical release of Wasmtime which supports [wasi-threads], such as
  Wasmtime 36 which is an LTS release.
* Use an alternative engine which implements [wasi-threads].

Wasmtime 36 will fall out of support in August 2027, about 16 months from now,
at which point WASIp3 cooperative threads is expected to be supported. It's
unlikely that the [shared-everything-threads] proposal will be a suitable
replacement by that point.

# Proposal
[proposal]: #proposal

The deletion of [wasi-threads] and `wasi-common` will be phased over a few
Wasmtime releases to give users a chance to be made aware of the changes that
are coming. Specifically for Wasmtime 45, releasing May 20, 2026, the
`wasi-common` crate will be marked as `#[deprecated]`. Usage of `-Sthreads` on
the CLI will issue a warning saying that it will become a hard error in a future
release.

For Wasmtime 47, releasing July 20, 2026, the `wasi-common` crate will be
deleted as well as the `wasmtime-wasi-threads` crate. The `-Sthreads` CLI flag
will yield an unconditional error indicating that it is no longer supported.

For Wasmtime 48, releasing August 20, 2026, the [wasi-threads] implementation
will not be present. This is Wasmtime's next LTS channel that will be supported
until August 20, 2028, and maintainers would like to minimize maintenance of
[wasi-threads] and keep it contained to the Wasmtime 36 release.

Note that Wasmtime 36 is a supported release of Wasmtime until August 20, 2027,
meaning that users who need time to transition can use this version of Wasmtime
while still receiving security fixes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Much of the rationale for this proposal is located in the
[motivation](#motivation) section above, but the overall intent of this RFC is
to make Wasmtime's implementation reflect reality.

* The [wasi-threads] proposal is now considered legacy and receiving no further
  development, and Wasmtime's implementation of it is known to be unsound and
  have various limitations.
* The `wasi-common` crate is a large, legacy crate in Wasmtime's codebase which
  has a growing maintenance that is only increasing over time. There is no use
  for `wasi-common` outside of [wasi-threads].
* The [shared-everything-threads] proposal is expected to be the successor to
  [wasi-threads] and is expected to solve many of the fundamental problems of
  [wasi-threads].

A theoretical alternative to this proposal is for someone to step up and
champion the [wasi-threads] proposal, but this is not realistically viable as
it's effectively been concluded already that trying to advance [wasi-threads]
eventually leads to the [shared-everything-threads] proposal. The
[shared-everything-threads] proposal is such a different direction than
[wasi-threads] that there's virtually no benefit in keeping the existing
implementation and/or trying to incrementally improve it.

Improvements to Wasmtime's implementation of shared memory are of course
welcome, and this can increase the viability of maintaining an external
implementation of [wasi-threads]. This alternative would require an interested
party to step up and perform this maintenance, however.

# Open questions
[open-questions]: #open-questions

None at this time.

[wasi-threads]: https://github.com/webassembly/wasi-threads
[threads]: https://github.com/webassembly/threads
[shared-everything-threads]: https://github.com/webassembly/shared-everything-threads
