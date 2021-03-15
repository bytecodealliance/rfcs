# Summary
[summary]: #summary

This RFC proposes to transition the default x86-64 compiler backend in
Cranelift's `codegen` crate, and also in Wasmtime when using the
Cranelift backend, to the "new", "MachInst", or "VCode" backend that
has been developed over the past year. Given that it has achieved
sufficient feature-completeness to pass all relevant tests, has
undergone testing in CI and via fuzzing since late 2020, and has
recently passed a security audit, we believe it is time to transition
the default setting. This RFC will propose a set of criteria that we
should ensure are met before we actually make the switch, and a set of
steps to take once we do. Note that this RFC does *not* propose
removing the old backend; it should remain an option that is
selectable both by a runtime "variant" flag, and by a build-time Cargo
feature flag that makes it the default variant again.

# Motivation
[motivation]: #motivation

For the past year, we have been developing a [new backend
infrastructure](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift/codegen/src/machinst/)
for Cranelift. Although this infrastructure was initially brought-up
in tandem with our aarch64 backend, we soon began work on a new x86-64
backend in the framework as well, since our long-term plan was to
deprecate and eventually remove the old Cranelift backend design and
related infrastructure.

For the past year, the current ("legacy" or "old") x86-64 backend has
remained the default for a user of the `cranelift-codegen` crate, or
`wasmtime`, who does not set any non-default options. However, we have
ensured that the new backend runs on CI alongside the old one, and it
is available under a feature flag today.

Work on this new backend has progressed to the point that it is now
stable, generally has good performance, and seems to be generally
easier to work with and maintain. Furthermore, we have gained
confidence in its security stance after a security-focused evaluation.

Given the increasing confidence in the new x86-64 Cranelift backend,
several significant users of the compiler have switched to the new x86
backend by default. In particular, Lucet ([PR
#646](bytecodealliance/lucet#646)) and rustc\_codegen\_cranelift ([PR
#1127](bjorn3/rustc_codegen_cranelift#/1127) and [PR
#1140](bjorn3/rustc_codegen_cranelift#1140)) have already switched to using the new backend by default (Lucet) or as the only option (cg\_clif).

Given this setting, we believe it is time to consider switching the
default backend for the `cranelift-codegen` crate, hence for all users
by default; and also, at the same time, for Wasmtime.

This discussion began in summer 2020 in issue
[#1936](bytecodealliance/wasmtime#1936); this RFC extends @bnjbvr's
initial proposal and excellent fact-finding work in that issue in
order to get final sign-off and finish the transition.

# Proposal
[proposal]: #proposal

At a high level, we propose (i) evaluating criteria by which we decide
to make the transition; (ii) obtaining the appropriate sign-offs; and
(iii) transitioning in several steps in a way that is designed to
preserve stability and leave options if anything goes wrong.

## Transition Criteria

1. *Feature-completeness*: All needed functionality must be present and
  all tests must pass. 

  -  Status: we are largely there already. In
     bytecodealliance/wasmtime#2718, a trial/draft PR, the default was
     switched and all tests that are still applicable (i.e., not testing
     specific details of the old backend) are passing. Only
     bytecodealliance/wasmtime#2710 needs to land for the new backend to
     be fully feature-complete.

     - Note: the new backend does not fully support Wasm-SIMD. However,
       the old backend no longer does, either, after the proposal's
       recent evolution. Hence, we are willing to accept this
       incompleteness because it is not a regression overall.
   
2. *Performance*: The new backend has acceptable compilation speed and
  generates acceptably good code.
  
   - Earlier evaluations showed that we were trending this way. No
     recent comprehensive comparison has been done, however. This RFC
     proposes to use the
     [Sightglass](https://github.com/bytecodealliance/sightglass/)
     benchmark suite in order to evaluate both dimensions.
   
   - Compilation time: we should expect as good or better compilation
     time on most benchmarks. We have seen some cases where the register
     allocator in the new backend can be slower on very large
     inputs. This RFC proposes to balance such slowdowns against the
     other benefits of the transition and to cautiously accept a small
     budget for some such cases, in tandem with a priority effort to
     understand the slowdown and address it soon.
   
     - Open question: how much degradation should we accept?
   
   - Runtime: we should expect as good or better code generated on most
     benchmarks.
   
     - Open question: how much degradation should we accept?

3. *Compatibility*: there should be no known or open blocking issues
   with the new backend when used by any significant user; in other
   words, we should not break anyone, and if the transition would do
   so, we should work with these stakeholder(s) first to work around
   the issue.
  
   - Status: cg\_clif and Lucet have already transitioned to the new
     backend. Wasmtime is compatible. Firefox uses Cranelift only on
     aarch64, which is already using the new backend framework.

     - Open question: are there other projects with which we should
       consult?

4. *Clean fuzzing record*: we should transition our fuzzers to use the
   new backend exclusively, and wait to ensure that no issues arise. We
   are currently fuzzing the new backend in Wasmtime differentially
   against an interpreter (`wasmi`), and in the past we have fuzzed it
   differentially against the old backend when embedded in
   Lucet. However, we have other fuzz targets as well that drive the
   compiler in different ways.

## Transition Steps

1. Switch the fuzzers to use the new backend. Add the appropriate
   feature flags to `fuzz/Cargo.toml`; wait and verify that `oss-fuzz`
   is running the new builds and that no issues arise.
   
   - *Open question*: how long should we wait?
   
2. Make a final Wasmtime/Cranelift release with the old backend as
   default, to provide the latest possible "new features on old
   backend" snapshot.
   
3. Land bytecodealliance/wasmtime#2718, which switches the defaults
   and makes the old backend a non-default option.
   
   - All users of `cranelift-codegen` and `wasmtime` get the new backend by default.
   
   - Both backends are built into the crate. Any user can
     programmatically ask for `BackendVariant::Legacy` when
     instantiating the compiler backend.
     
   - The filetest infrastructure accepts variants: `target x86_64
     legacy` vs. `target x86_64 machinst` in test files, and
     instantiates the appropriate backend.
     
   - By depending on the codegen or wasmtime crate with the
     `old-x86-backend` Cargo feature enabled, the old backend becomes
     the default again when no backend variant is specified.
     
   - The old backend continues to be tested on CI in a separate job,
     just as the new backend is tested today.
     
4. Actively monitor users and ensure that no breakage occurs. Address
   any issues by fixing bugs, implementing any functionality we have
   missed, or helping users to select the old backend, if appropriate.
   
5. Make several releases with the new backend as default. Keep the old
   backend alive with CI; it will be "fully supported" during this
   time, but its use will be deprecated.

6. At some future point, when we are confident that no significant
   use-cases or dependencies remain, we can remove the old
   backend. This will be a separate RFC process with its own
   consensus-gathering.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This transition's rationale largely derives from the rationale for the
new backend in general: the new design is simpler, more maintainable,
generally more performant, and has a more trustworthy security
stance. Given these benefits, if there are no blocking issues, it
makes sense to switch the default.

Furthermore, there is a continuing cost to maintaining support for the
old backend. Its presence in the codebase adds complications because
many data-structures and code paths have to account for both
cases. When we add features that require backend support, we have to
evaluate a tricky tradeoff question: do we spend time adding the
feature to the old backend as well, given that at some future point it
will be removed? If such work is necessary to support a feature in the
*default* build today, then extra work is required that will
eventually no longer be useful.

Because of this cost and the tradeoffs of spending more effort on the
old backend, it is already stagnating slightly in terms of feature
support. For example, its SIMD support is not up-to-date with respect
to the latest Wasm SIMD proposal. As other Wasm proposals advance, it
will likely fall further behind and time spent updating it will be
harder and harder to justify.

The major alternative is simply to do nothing: retain the old backend as
default, and new backend as a non-default option. This is the status-quo and
has the least risk; individual embedders can always enable the new backend if
desired.  However, it has the significant downside that it implies ongoing
support for the old backend; in the steady state, this is twice the maintenance
and possibly feature-implementation work for one platform (x86-64), and so we
do not consider this to be a serious option *unless* the new backend shows
serious defects that prevent switching.

In other words, given the direction that effort is being, and will be, spent,
and given the effort and maintenance burden of various options in the future,
it seems to be inevitable that the switch will occur at some point. The
question to answer is whether the new backend is ready *yet* to be the default,
or whether we need to wait longer.

# Open questions
[open-questions]: #open-questions

1. How much compile-time and runtime performance degradation should we
   accept, if any, before moving forward with this transition?
   
2. Which other projects or users should we consult to ensure that we
   will not cause unnecessary breakage?
   
3. How long should we allow the new backend to run on all fuzz targets
   (not just the new-backend-specific one that already exists) before
   moving forward?
   
4. How long should we retain the old backend before starting to
   consider its removal?

5. Should we audit the old backend's code or tests to see if we are
   missing any significant functionality or test coverage that should
   be moved over?
