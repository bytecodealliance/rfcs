# Summary

[summary]: #summary

This RFC is a proposal to release Wasmtime with a 1.0 version. In doing so this
proposal also defines what it means to be 1.0 and details areas such as:

* What is Wasmtime's stability story.
* What will Wasmtime's release process be.
* What users can expect from Wasmtime going forward.

# Motivation
[motivation]: #motivation

Wasmtime has for quite some time been "production ready" in the sense that it's
actually used in production and the technical implementation is reliable enough
to use. There are a number of points which may cause users to be hesitant to use
Wasmtime in production, however, including:

* Wasmtime's literal version number is in the "pre-release" phase (it's 0.29.0
  at the time of this writing). The lack of digit in the "major" spot can deter
  folks by giving off a perception that Wasmtime isn't ready for
  production use yet. While this is somewhat normal in the Rust ecosystem to
  have battle-hardened crates in the 0.x.y phase, this is much less normal in
  the broader software-in-general ecosystem.

* Wasmtime does not have a clear commitment to stability at this time. We have
  the RFC process for big changes but it's not clearly defined otherwise when
  and where Wasmtime may change its APIs. This means that users considering
  Wasmtime for long-term use don't have a good idea of what to expect from the
  Wasmtime project in the long-term in terms of maintenance and updates.

* Wasmtime does not currently have a formal release schedule or process.
  Releases are currently somewhat ad-hoc "when we feel like it". While this has
  worked well enough for now it's not necessarily good enough to rely on since
  users don't know when to expect new features to be released or what the
  release cadence might look like.

* Wasmtime does not have a well-defined process for which releases are supported
  and which receive fixes. For example with [CVE-2021-32629][cve] we backported
  fixes to the previous stable release of Wasmtime, but not others. This was
  done in an ad-hoc and not well-defined manner, and isn't the best way to do
  this moving forward.

The motivation for this RFC is to effectively fix all these issues. Wasmtime is,
as a codebase, technically ready for use in production. This RFC intends to
cover the rest of the ground necessary to make Wasmtime suitable for production
use, mostly on process-aspects and such. Once this RFC is implemented we, as
project maintainers, and users, as developers using Wasmtime, should all be
confident that Wasmtime feels "1.0" and is production ready in all aspects.

Many projects in the open source ecosystem are "production ready" and can serve
as inspiration to draw from for each of the above points. Release processes for
various projects can widely differ and serve different goals for both
maintainers and users. What's appropriate for one project doesn't necessarily
mean that it's also appropriate for Wasmtime itself. This proposal upholds a
number of goals that are specific to Wasmtime itself and how it's expected
Wasmtime will be used:

* Wasmtime should be released to users in a predictable fashion. For example
  "when will this feature be released?" should be easy to answer.

* Users should have a clear expectation for support of older versions of
  Wasmtime. For example "does this version get bug fixes and security fixes?"
  should be easy to answer.

* Users should be able to clearly understand what it means to keep up with
  Wasmtime and its development, and users should be able to easily upgrade
  between Wasmtime versions. For example "how do I get the latest version?" or
  "how do I upgrade from my current version?" should be easy to answer.

* Developers of Wasmtime should not be hindered by processes to land
  improvements to Wasmtime. For example "when can I land my 10% perf increase
  that tweaks APIs" should be easy to answer, and the answer shouldn't be
  "six months from now". Furthermore bugfixes which may break APIs should also
  not be difficult to land.

* Users should be able to clearly understand what version of Wasmtime they're
  using. It's possible to use Wasmtime through a variety of means such as the
  numerous embedding APIs for each language, and it should always be clear what
  version of the project you're using.

The constraints here do not map perfectly to what other projects do all the
time. For example Wasmtime will not have as strong of a committment to API
stability as the Rust language and standard library. Wasmtime does want,
however, to maintain the "largely hassle-free upgrade" experience inspired by
Rust's example, though. This proposal attempts to identify the right balance
between these various concerns for the Wasmtime project itself.

[cve]: https://github.com/bytecodealliance/wasmtime/security/advisories/GHSA-hpqh-2wqx-7qp5

# Proposal
[proposal]: #proposal

A comprehensive 1.0 release story for Wasmtime comprises of a number of
inter-related parts, many of which can greatly affect how the others are
organized. This proposal is broken down into a number of sub-parts, but it's
recommended to read through all of them to get a good idea of how they're all
related.

## Stability and Future Development

Wasmtime is an implementation of a WebAssembly runtime. The actual
implementation of Wasmtime is ever-changing and ever-improving as better ways of
implementing previous constructs are discovered. Furthermore WebAssembly itself
is ever-changing and ever-improving where there are almost always active
proposals in flight to extend and modify WebAssembly. The nature of Wasmtime and
its goals mean that **Wasmtime's stability story will not include API stability
in the literal sense of "Wasmtime will always be 1.0 and never 2.0"**.

For Rust users this may feel a bit discombobulating where Rust is likely to
always be "1.x". This isn't viable for Wasmtime: the addition of new WebAssembly
features will often require breaking API compatibility, so we don't have much
choice in the matter. The ability to break API compatibility also enables
improvements to the runtime that wouldn't be possible without changes to the
embedding API. These are seen as critical abilities needed to improve Wasmtime
over time. As mentioned in the motivation section, Wasmtime developers themselves
should not feel overly burdened or having to contort their code to land changes
which are seen as important. Allowing breaking changes is seen as key to enabling
this.

Naturally, though, the world is not entirely made up of Wasmtime developers.
Instead it's much more likely that the number of users of Wasmtime far
outweighs the number of developers on Wasmtime. A stream of constant breaking
changes makes it feel harder to use Wasmtime since it can feel more difficult to
keep up with the latest and receive bug fixes and new features. Wasmtime's
stability story, consequently, will not be on the end of the spectrum where
breaking changes are landed willy-nilly. This means that there's a tension set
up between developers of Wasmtime and users of Wasmtime, but this is to be
expected from the stable release of a project.

On a spectrum of "Wasmtime never breaks any API ever" to "Wasmtime breaks all
its APIs every release" Wasmtime will reside towards the first half here. In
other words Wasmtime developers will be expected to rarely and deliberately
break existing APIs, but will also be allowed to break APIs as necessary. The
major goal here is "hassle-free upgrades" inspired by the Rust release process
where API stability in the absolute literal sense is detrimental to the
development of Wasmtime, but deliberate effort and work is done to ensure
upgrades are as seamless as possible.

Users should expect that they will regularly need to increase the major version
of Wasmtime to upgrade Wasmtime usage in their projects. Furthermore users
should expect that between upgrades they may need to tweak some APIs and update
them to continue to use the latest-and-greatest Wasmtime. At the same time
though users should expect that they will not be blindsided by major
refactorings. For example changes like [Wasmtime's new API][new-api] will still
go through the RFC process and will be deliberately scheduled and widely
advertised before they're released. Note also that changes of that magnitude are
expected to be quite rare.

[new-api]: https://github.com/bytecodealliance/rfcs/blob/main/accepted/new-api.md

## What is being stabilized?

Wasmtime is a relatively large project at this point with lots of components.
Additionally not everything lives in the `wasmtime` repository itself but there
are separate dependencies such as `wasm-tools`, `wasmtime-*` embeddings,
`witx-bindgen`, etc. This proposal does not attempt to blindly label everything
with 1.0 and ship it, but rather discrete "tiers" of support are defined for
what it means to use a component of the Wasmtime project.

At a base level anything shipped in Wasmtime is guaranteed to clear a minimum
threshold of quality. All dependencies, even the transitive ones, used by
Wasmtime are guaranteed to be "production quality" for the surface area that
Wasmtime uses. On the other hand, though, not all of Wasmtime's dependencies
and components will meet the same level of API stability expected of a stable
project. For example Cranelift is not an API-stable component that will be
released with Wasmtime. For now Cranelift will likely have the same release
cadence of Wasmtime for ease of infrastructure, but this may change in the
future as Cranelift develops its own API and release process. Additionally,
though, the internal crates of the `wasmtime` implementation, such as
`wasmtime-runtime`, are not intended to ever be API-stable but are intended to
be production quality.

Public-facing components of Wasmtime that users are expected to use will be
classified into three different tiers of support. These tiers are defined by the
stability of the API users can expect as well as the quality of implementation
they can expect. The tiers are organized into the best-supported to
least-supported order, with components in tier 1 being the best supported. Note
that as mentioned above internal dependencies of Wasmtime, such as Cranelift,
`wasmtime-*` internal dependencies, or `wasm-tools` crates, are not classified
in these tiers because they are not intended to be public-facing parts of the
Wasmtime project that users use.

#### Tier 1 - API stable, production quality

The highest tier, similar to Rust's "tier 1 platforms" is defined as the highest
level of support and stability for Wasmtime components. Components that are tier
1 are expected to at least meet this criteria:

* API stable - this does not mean "forever stable" but it does mean that major
  breaking changes are rare.
* RFCs required for major changes - all large changes to the API or
  functionality will require an RFC to be approved.
* Well-maintained - at least one member of the Bytecode Alliance is actively
  maintaining this component and providing it with security updates, bug fixes,
  and new feature development.
* Released with Wasmtime - these components are all released on the same cadence
  as the rest of the Wasmtime project (more on this cadence later).
* Production quality - this component is well vetted and reviewed by multiple
  developers. Landing changes requires code review and extensive test suites to
  pass.

At the time of this RFC, the only component in the Wasmtime ecosystem which
meets these criteria is the `wasmtime` Rust crate itself. This tier is intended
to be quite strict in terms of requirements and nontrivial to reach, and the
other components of Wasmtime today do not yet currently meet all of these
requirements and consequently will fall into lower tiers.

#### Tier 2 - API unstable, production quality

This second tier of support differentiates itself from tier 1 components not
requiring as much API stability. The same level of production quality for code
and implementation, however, is expected of tier 2 components. The checklist for
this tier is:

* API unstable - strict API stability is not required at this time. Components
  can freely have major changes to their API without much warning. This is
  intended to be an incubation area for production quality implementations which
  haven't yet necessarily had enough experience to determine the best API yet.
  Additionally RFCs are not required for changes in this tier.
* Well-maintained - same as tier 1.
* Released with Wasmtime - same as tier 1.
* Production quality - same as tier 1.

Wasmtime components that fall into this category are the C API of Wasmtime and
the `wasmtime-wasi` crate. In both of these cases the API stability isn't quite
there yet, but both are well-reviewed and production quality.

Note that no language bindings are yet to the Tier 2 level yet. This is
intentional and discussed in the next section.

### Tier 3 - API unstable, not guaranteed production-ready

This is the lowest of the tiers of supported APIs for Wasmtime and is intended
to be a sort of "catch-all" for everything that doesn't fall into the above
tiers. This tier does not come with many guarantees associated with it, and is
intended to be a breeding ground and call-for-help for components to reach tier
2 or higher status.

Components in tier 3 may still meet various requirements of tier 1 and 2 while
still being classified as tier 3. For example components here may not have code
review but could still be of a high-enough quality to use in production.
Similarly they might be released at the same cadence of Wasmtime itself but
lacking in API stability.

Today this tier includes:

* `wasmtime-py` - Python bindings for Wasmtime
* `wasmtime-go` - Go bindings for Wasmtime
* `wasmtime-dotnet` - .NET bindings for Wasmtime
* `wasmtime-cpp` - C++ bindings for Wasmtime
* `witx-bindgen` - Canonical API bindings generator for Interface Types

Each of these projects is missing at least one criteria from the list of Tier 2
requirements. As is inherent to this tier, however, this is a call-for-support
for others who are interested in moving these projects up a tier of support. For
example most of these projects primarily need more maintainers to help review
bindings for language idioms and implementation, and that's all that's necessary
to move into tier 2.

It's also worth highlighting that the intent here is to establish a deliberately
high bar of quality we commit to. In many established open source projects, the
level of quality, and the development and reviewing practices some or all of
these components have and use would be deemed "good enough". And in many domains
they might be. We're acutely aware of Wasmtime's, and more generally
WebAssembly's use in mission- and security-critical environments, and we believe
that we need to hold ourselves to higher standards accordingly.

## What does it mean for a feature to be stable?

The tiers of support for Wasmtime are primarily concerned with the APIs that
users interact with and the support expected there, but Wasmtime is also
defined by the stability of its implementation. Features implemented in Wasmtime
itself are held to a high standard of quality which should fit into the
quality expectations of components that are tier 1. Features enabled-by-default
and implemented in Wasmtime are expected to meet some baseline criteria:

* The feature must be throughly tested in Wasmtime's CI on major platforms (at
  this time x86\_64 and AArch64)

* If the feature is for a WebAssembly upstream proposal, all spec tests must be
  enabled and passing and the proposal must be in [stage 4 or
  later](https://github.com/WebAssembly/meetings/blob/main/process/phases.md).

* The feature must have no open questions, design concerns, or serious known
  bugs.

* The feature must have support in fuzzers and have been fuzzed for at least a
  week. Additionally we should be confident that the fuzzing support for this
  feature is exercising the necessary bits in Wasmtime thoroughly.

* The feature is supported in the Rust API, the C API, and at least one other
  language's embedding API.

* A member of the Bytecode Alliance must be "on the hook" for maintenance of
  this feature.

Features can be implemented in-tree even if they do not meet these criteria but
the features must be disabled-by-default at either runtime or compile
time. If a feature's interim implementation does not have an undue compile-time
or runtime footprint then it can be off-by-default at runtime but compiled in by
default. If, however, an in-progress feature has a significant compile-time or
runtime footprint it must be disabled at compile-time by default.

Features implemented in-tree but not currently stabilized are also subject to
removal if there is no active progress being made on the feature. For example if
this RFC is approved **the `lightbeam` feature of Wasmtime would be removed**.

> **Note**: This RFC proposes removing `lightbeam` because it isn't actively
> maintained, and as a result hasn't been compiling successfully for quite
> some time. The Wasmtime project is still interested in the addition of a
> baseline compiler, but the addition of one would be proposed in an RFC and
> undergo careful evaluation to ensure the design aligns with Wasmtime's
> goals and requirements, and the compiler has a solid maintenance story.

## Release Process and Cadence

Wasmtime intends to follow in the footsteps of many other projects on the matter
of release cadence with a frequent and predictable release cycle. **This RFC
proposes releasing Wasmtime on Tuesday every 4 weeks**. The precise date of
each release may be adjusted to avoid coinciding with public holidays, though,
which could result in some releases being a slightly different width apart.

The goal is to publish work and improvements in Wasmtime on a relatively rapid
schedule to ensure that the latest-and-greatest is available for usage. This is
also intended to signal to users that consumers of Wasmtime should expect to
stay up-to-date with Wasmtime and it's not appropriate to ossify a version for
years and expect it to continue to work and receive everlasting maintenance.

Each release of Wasmtime will bump the major version number of Wasmtime itself.
This means, for example, that Wasmtime 2.0 will be released four weeks after
Wasmtime 1.0. After one year Wasmtime will be at 14.0. At this time it is not
planned that "minor" releases will be made of the 1.1.0 variety. Note that this
means means that not all major releases will actually contain breaking changes.
The reason we propose to nevertheless always bump the major version is that this
allows us to keep Wasmtime's version number in sync with that of its various
language-specific embeddings: as a developer using Wasmtime, you shouldn't have
to worry about how the version number of your language's Wasmtime embedding
lines up with that of Wasmtime itself. E.g., if you use `wasmtime-py 7.0`, you
can be sure that you're using Wasmtime 7.0. Trying to keep major version bumps
to a minimum while keeping version numbers aligned would force us to still bump
Wasmtime's and all embeddings' version numbers whenever there's a breaking
change in even a single language-specific embedding. As noted in the previous
section, though, each release is expected to be a relatively hassle-free
upgrade, so while API-breaking changes are allowed they're not necessarily
encouraged.

Wasmtime will continue to create a tag for all released versions of Wasmtime
with a corresponding GitHub release as is done today for the CLI and C API
binaries. Embeddings will all be tagged and released as appropriate to
language-specific package managers (such as crates.io and PyPI). Wasmtime will
always be released from the `main` branch of the Wasmtime repository itself,
which means that all development is happening on `main` and once something lands
it's guaranteed to be in the next release, unless reverted. Note that bug fixes
and such for historical releases are discussed later in this proposal.

Releasing a new version of Wasmtime every 4 weeks can be quite rapid for some
users who don't necessarily want to stay up-to-date with the latest and greatest
of Wasmtime, but still want the stability of a production-ready WebAssembly
runtime. For these users, this leads well into the next section ...

### Long-term Support Releases

Wasmtime will support some releases for an extended period of time relative to
other releases. These releases will be known as "long term support" releases or
LTS releases. **Wasmtime will support two active LTS versions at any point in
time, each supported for 10 releases at a time**.

For users who want to use Wasmtime but are not interested in upgrading monthly,
this will be a suitable alternative where Wasmtime LTS versions will have the
following properties while the version is considered "supported":

* They will receive security fixes.
* They will receive bug fixes related to correctness.
* They will _not_ receive new features.

For example if a new WebAssembly proposal is implemented it will not be
backported to LTS versions. Additionally if a bug is fixed in a wasm proposal
that is off-by-default in an LTS version it will also not be backported. Other
bug fixes (and of course security fixes) will be backported to supported LTS
versions.

An LTS version is considered "supported" for 10 release cycles, or 40 weeks (~9
months). Wasmtime will have two LTS versions at any point in time, with a new
LTS happening every 5 releases (20 weeks, ~5 months). For ease of remembering
what's an LTS and what isn't, all releases of Wasmtime divisible by 5 will be
LTS version, with the exception of 1.0 being an LTS version as well. For example
the LTS versions of Wasmtime will be 1.0, 5.0, 10.0, 15.0, etc.

This cadence means that users who do not want to upgrade Wasmtime monthly will
be expected to upgrade Wasmtime at least every 9 months, likely every 5 months.
These upgrades are likely to be less "hassle-free" than each individual version
upgrade since it will accumulate at least 5 releases worth of minor breaking
changes. But in exchange, users of LTS versions have a 5-month window to
upgrade while staying on a supported version.

LTS versions will be maintained in separate branches of the Wasmtime repository.
At this time it's expected that LTS versions will not needed separate branches
in separate embedding repos and tags will suffice. The branch names for the
Wasmtime repository will be `lts-latest` and `lts-oldest` for the most recent
and the second-most-recent LTS version.

## Backports - Security fixes

With an established concept of releases and LTS for Wasmtime this provides a
framework to discuss how security issues are handled in Wasmtime's release
process. **Security issues will be applied to the current version and supported
LTS versions of Wasmtime**. Security fixes will always be released as patch
releases. The current version of Wasmtime at the time of the issue being made
public will be patched in addition to the LTS versions at the time.

For example, if Wasmtime is currently at 12.0 then the current LTS versions are
5.0 and 10.0. If a security issue is identified at this time then the following
new releases will be made available: 5.0.1, 10.0.1, 12.0.1. No other versions of
Wasmtime is guaranteed to receive a patch, for example 11.0 will not be patched
as it's neither LTS nor current. Additionally 1.0 will also not be patched
despite it being an LTS version because it is no longer a supported LTS
version. Some older releases may be patched at the discretion of the project at
the time of the release, however.

Patch releases in this sense are expected to be *guaranteed* to be a low-effort
upgrade. Wasmtime developers will ensure that 5.0.1 is API-compatible with 5.0.0
in the literal Semver sense (for all embeddings, not just the Rust embedding).
In other words, users should expect upgrading to a patched release of Wasmtime
to be trivial to perform.

## Backports - Bug fixes

While not as critical as security issues bugs do happen and fixes will get
landed. **Wasmtime will adopt a policy where a bug fix is backported to the
current version and supported LTS versions if it fixes on-by-default behavior
and is suitable to backport**. This should ensure that both the current version
and LTS versions are free of known-bugs for on-by-default behavior.

The two clauses about about bug fix backports are:

* Bug fixes are only candidates for backporting if they fix on-by-default
  behavior. For example at the time of this writing Wasmtime doesn't have the
  SIMD proposal of WebAssembly turned on by default, so bug fixes for the SIMD
  proposal would not be backported. Bug fixes related to bulk-memory, however,
  would be backported.

* To be backported bug fixes also need to be suitable to backport. Fixes such as
  performance improvements do not fall in this category, but the main members of
  this category are "fixes behavior to align with the official WebAssembly
  specification". For example if a wasm module may not exhibit the behavior
  defined in the spec for any reason, then that bugfix is candidate for a
  backport.

## Backports - Release Process

When a bug fix or a security fix is backported this means that a new release of
Wasmtime needs to be made. **Security fixes will immediately be accompanied with
patch releases** at all times. Bug fixes will be released on the same monthly
cadendce as other releases.

For example if a bug was discovered two weeks after 6.0 was released, then the
bug will be backported to the 1.0 and 5.0 versions. When 7.0 is released
Wasmtime would release 1.0.1, 5.0.1, and 7.0.0. Bug fixes, unless serious
enough, are not released to the current version (e.g. no 6.0.1).

If a bug is serious enough to warrant one, however, it will be accompanied with
an immediate release like security fixes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Designing a release process and stability story for Wasmtime involves a fair
amount of subjectivity and naturally there are a number of alternatives that
could be pursued beyond just tweaking minor details of the proposal:

* One immediate alternative is to attempt to provide strong API-stability
  guarantees instead of a possibly-API-breaking change every 4 weeks. The reason
  this was not proposed is that it's expected that it would unnecessarily hinder
  development of Wasmtime itself. The implementation of any new WebAssembly
  proposal is highly likely to alter the public API of Wasmtime in some way
  that is breaking, so such a proposal for string API-stability guarantees would
  need at least some concession for when to do breaking changes. The most likely
  form of this would be to batch up breaking changes, perhaps in an LTS-style
  cadence. This means, though, that new features are artificially delayed in
  Wasmtime and cannot land when they are originally implemented. Furthermore
  there would be no way to land intermediate work which would continue to be
  developed in-tree. Overall it's expected that users who want API-stability
  from Wasmtime are sufficiently serviced with Wasmtime's LTS versions which
  guarantee API-stability but lack new features. At this time an alternative of
  API-stable releases still getting new non-API-breaking features is seen as too
  much of a hindrance to the development of Wasmtime itself.

* Technically there's nothing really stopping feature-development being
  backported to LTS versions so long as the feature doesn't change any APIs.
  This proposal, however, only indicates that security issues and bugs are
  backported to LTS versions. This is intended to be a relatively conservative
  starting position where we could still backport features as necessary if
  someone's willing to put in the work. The fear, though, is that the backport
  process becomes relatively complicated and LTS backports are likely less
  battle-tested than changes on `main` due to the nature of mismatch between the
  original state of `main` and the LTS itself. To help make what is already a
  somewhat complicated process a bit simpler, this proposal states that new
  features are not backported to LTS releases.

* LTS versions in theory could be dropped entirely. There aren't known users who
  are specifically asking for LTS versions and existing production users of
  Wasmtime are likely to stay with released versions to get improvements as they
  come out. This proposal assumes, however, that there are developers we're not
  actively hearing from who would benefit from such LTS versions and we'd like
  to accomodate them. If in practice it turns out that there is very little
  usage of LTS versions we can consider dropping support for them at a later
  date when such a conclusion is made (probably another RFC).

* As currently proposed Wasmtime is guaranteed to receive a new major version
  every 4 weeks. It might be the case, though, that within those 4 weeks of
  development that Wasmtime didn't actually have any API-breaking changes occur.
  Theoretically this could allow for a minor release of the form 2.1.0, for
  example, if 2.0.0 was the previous release. There are a number of downsides
  with trying to do this, though, including:

  * Wasmtime developers now have to keep track of breaking changes and determine
    whether any landed in the 4-week release window.
  * This makes Wasmtime's versioning less predictable.
  * LTS is somewhat more murky in this scenario. If it still happens every 5th
    release we'll have to keep track of what the 5th release is, and if both
    2.0.0 and 2.5.0 are LTS releases then it's not clear why projects would use
    2.0.0 as opposed to 2.6.0 if it were the current version (since everything
    is semver-compatible).
  * If the `wasmtime` Rust crate didn't have any breaking changes it doesn't
    mean that the embedding APIs also didn't have any breaking changes (or vice
    versa too).

  By having a new major version every 4 weeks it's definitely a form of "least
  common denominator" solution but it also provides a predictable versioning
  number as well as a clear indication of what is LTS and what isn't. For these
  reasons it's proposed here to use a new major version every 4 weeks instead of
  gauging via breaking changes or not.

# Open questions
[open-questions]: #open-questions

- What sort of automation is required for Wasmtime developers to keep up with
  once-a-month releases? The process described here is relatively expansive and
  can take quite some effort to keep up with. For example:

  * Can we entirely automate releases?
  * Could this automation also get connected to all the other embeddings and
    repositories?
  * How hard will it be to make a patch release with a bug fix?
  * How hard will it be to remember what branches need backporting?

- Will living on an LTS release be annoying to users? For example users may not
  know that each 5 releases are LTS or they may otherwise get warned by tooling
  like `cargo outdated` that their dependency is behind-the-times when it's
  intentionally so. One possibility to solve this would be `wasmtime-lts`
  packages for all package managers which are versioned independently from the
  `wasmtime` package (basically the `wasmtime` package divided by 5). In Rust,
  for example, the crate would simply reexport the `wasmtime` package itself.
  This would require a `*-lts` release of all Wasmtime crates, though, including
  those like `wasmtime-wasi` and it's not sure if this would work well from an
  automation point of view and such.
