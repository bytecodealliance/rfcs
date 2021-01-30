# Summary
[summary]: #summary

This RFC proposes setting up a benchmarking and performance-tracking
infrastructure to track Cranelift and wasmtime performance over time. We would
like to (i) establish some server infrastructure, (ii) build automation
integrated with our GitHub repos, and (iii) provide a web-based frontend to
show historical and current performance data.

# Motivation
[motivation]: #motivation

As we develop the Cranelift compiler and its machine backends, and as it is
used in more production settings, its performance becomes increasingly
important. In many settings, a few percentage points change in compilation time
or generated-code performance is quite a big deal. In addition, experience in
many compiler projects over the years has shown that small performance deltas
have a way of accumulating over time; if there is no systematic tracking, this
gradual drift can add up to sizeable performance regressions.

It's thus important both to track historical trends, in order to avoid
regressions and (more positively) to measure our progress; and to measure the
effect of individual changes, when significant enough, to determine objectively
whether optimizations or design choices are good ideas.

Independently of *what* benchmarks we measure, we need an *infrastructure* to
measure, record, and visualize statistics. This RFC proposes building such an
infrastructure.

# Proposal sketch
[proposal]: #proposal

## Step 1: Establish physical infrastructure

First, we need to establish a set of machines to run benchmarks and host
results. The GitHub-hosted CI infrastructure we currently use for testing will
likely not be adequate, because (i) runtime of benchmarks will be longer than
we would like for CI actions, (ii) we don't want to tie benchmark runs to CI
runs, necessarily, and (iii) CI-runner environments are unpredictable and
unreliable environments for accurate, low-noise benchmark measurements.

We already have access to one AArch64 server-grade host for interactive
development of the Cranelift ARM backend, generously [provided by
ARM](https://github.com/WorksOnArm/cluster/issues/204), and have the ability to
run benchmarking on it. We have also procured access to a dedicated x86-64
machine. These two machines would allow us to benchmark Linux/x86-64 and
Linux/aarch64. Another worthwhile target would be Windows/x86-64, which is
interesting because of differing ABIs and some VM details, but we have not yet
decided to commit resources (an additional machine) or develop specific runner
infrastructure for this.

Relevant Bytecode Alliance-affiliated engineers would then be granted access to
these machines and maintain a benchmark-runner infrastructure as described
below.

## Step 2: Build and host a metrics-visualization web UI

We will need a frontend to present the benchmarking data that this
infrastructure collects. Johnnie Birch and Andrew Brown of Intel have developed
a prototype web-based frontend that records and plots performance of benchmark
suites over time; this is the most likely candidate.

We hope to initially work out a static-file / client-side solution, which would
allow us maximal flexibility in hosting configuration. For example, the
benchmarking infrastructure could upload the latest data to a Git repository,
to be served by GitHub Pages. We do not anticipate the need for a full
database-based backend, and if data becomes large enough then we can segment
the static data (JSON) files and download selectively.

Initially, we can populate the data for this UI with manually-initiated
benchmarking runs on the benchmarking machines.

## Step 3: Set up GitHub-integrated benchmark runner automation

Once we have the machines set up, a UI established, and the ability to run
benchmarks and upload results, we should develop a daemon that runs
continuously on benchmark-runner machines and monitors our GitHub repositories.
When a request for a benchmark run is initiated, the runner daemon should start
a run; when the run is complete, it should either post the results on the
relevant GitHub PR conversation, or include the results in its web UI, or both,
as appropriate.

The largest design question will be how this automation ensures security:
because the runner will download a git commit and execute arbitrary code from
that commit, we *cannot* simply execute this on every PR or every anonymous
request. Rather, we will likely have a bot that includes an allow-list of
trusted project members, and wait for a special GitHub comment from such a
person to kick off a run.

If any of the above becomes impractical or difficult, an intermediate
design-point could involve a web interface that allows approved users
(authenticated in some way) to request a run on a given git commit hash; this
would not require any GitHub API integration. We expect, though, given prior
art in GitHub-integrated CI-like bots (such as the Rust project's bors bot),
that the integration should not present unforeseen problems.

Note that this runner infrastructure, at the macro level (repo checkout and
command invocation), is largely orthogonal to the benchmark suite and its
specific harness. We will likely want to design a simple in-repo configuration
file consumed by the runner daemon that specifies what to run and in what
environment (e.g., a Dockerfile-specified container).

# Open questions
[open-questions]: #open-questions

## Security

We need to ensure that, given the potential for arbitrary execution of code
from a PR, the necessary safeguards are in place to gate on approvals.
Potentially we should also sandbox in other ways; for example wrap the runner
daemon in a chroot or container with limited capabilities, and cap its resource
limits.

## Availability Expectations

We should set expectations that the benchmarking service may occasionally
become unavailable due to hiccups in operational details: for example, a
build/benchmarking server might go down or run out of disk space, or a PR might
break a benchmark run and not be caught by CI. At least two questions arise:
how or if such a situation blocks work from being done, and what resources
(time / engineering) we apply to minimize it.

Given that not every PR will be performance-sensitive and require benchmarking,
such downtime should not impact progress in most cases, so it is likely to work
well enough to start with a "best effort" policy. In other words, repairing any
benchmark-infrastructure breakage should be done, but is lower-priority than
other work, so as to not impose too much of a burden or require an "on-call"
system. If a performance-sensitive PR is blocked on a need for benchmarking
results, then we can expedite work to bring it back online. Otherwise, the
ability to request runs on arbitrary commits should allow us to fill in a gap
after-the-fact and reconstruct a full performance history on the `main` branch
even if we experience some outages.
