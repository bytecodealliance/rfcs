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
ARM](https://github.com/WorksOnArm/cluster/issues/204). We may be able to use
this host for running benchmarks as well, if all stakeholders agree.

We will need an x86-64 host as well; the best option is likely to rent a
low-cost dedicated host. Details of hosting choice and funding are TBD.

Relevant Bytecode Alliance-affiliated engineers would then be granted access to
these machines and maintain a benchmark-runner infrastructure as described
below.

## Step 2: Build and host a metrics-visualization web UI

We will need a frontend to present the benchmarking data that this
infrastructure collects. Johnnie Birch and Andrew Brown of Intel have developed
a prototype web-based frontend that records and plots performance of benchmark
suites over time; this is the most likely candidate. As long as traffic is low
enough, we can plan to use the benchmark-runner host(s) as web hosts as well in
order to serve this UI. Alternately, if hosting performance becomes an issue,
we could design a static/pregenerated-page workflow wherein the runners upload
data to a git repository and static data is served from GitHub Pages (or
another static host).

Initially, we can populate the database for this UI with manually-initiated
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

# Open questions
[open-questions]: #open-questions

## Security

We need to ensure that, given the potential for arbitrary execution of code
from a PR, the necessary safeguards are in place to gate on approvals.
Potentially we should also sandbox in other ways; for example wrap the runner
daemon in a chroot or container with limited capabilities, and cap its resource
limits.
