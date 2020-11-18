# Benchmark Suite for Wasmtime and Cranelift

[summary]: #summary

<!-- One paragraph explanation of the proposal. -->

We should create and maintain a suite of benchmarks for evaluating Wasmtime and
Cranelift. The benchmark programs should be representative of real world
programs, and they should be diverse such that they collectively represent many
different kinds of programs. Finally, we must ensure that our analyses are
statistically sound and that our experimental design avoids measurement bias.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected -->
<!-- outcome? -->

This benchmark suite has two primary goals:

1. Automatically detecting unintended performance regressions

2. Evaluating the speedup of proposed performance optimizations and overhead of
   security mitigations

## Automatically Detecting Unintended Performance Regressions

Without dilligent monitoring, unintended regressions will sneak past us. Manual
review is not sufficient, as the success of continuous integration tests shows
us. While our testing and fuzzing infrastructure will generally catch
correctness regressions, we have nothing in place for catching performance
regressions.

The benchmark suite should be run once per day (or as often as is feasible,
ideally it would run for every commit) and report any regressions in execution
time or memory overhead.

## Evaluating Proposed Performance Optimizations and Security Mitigations

The decision of whether to include a proposed optimization must balance
maintainability, compile times, and runtime improvement. Maintainability is
subjective, but our new benchmark should provide quantitative data for the
latter two: how does the new optimization affect compile times, and what kind of
speed ups can we expect? Does it shrink emitted code size or impose memory
overhead? The answers to these questions will help us evaluate a proposed
performance optimization.

Similarly, these same types of questions appear when evaluating a security
mitigation. How much overhead does, for example, emitting
[retpolines](https://software.intel.com/security-software-guidance/api-app/sites/default/files/Retpoline-A-Branch-Target-Injection-Mitigation.pdf)
add to WebAssembly execution time?

Experiments like these are invaluable for engineers and researchers working with
Wasmtime and Cranelift.

## Nongoal: Creating a General-Purpose WebAssembly Benchmark Suite

It is also worth mentioning this explicit nongoal: we do not intend to develop a
general-purpose WebAssembly benchmark suite, used to compare between different
WebAssembly compilers and runtimes. We don't intend to trigger a WebAssembly
benchmarking war, reminiscent of JavaScript benchmarking wars in Web
browsers. Doing so would make the benchmark suite's design high stakes, because
engineers would be incentivized to game the benchmarks, and would additionally
impose cross-engine portability constraints on the benchmark runner. We only
intend to compare the performance of various versions of Wasmtime and Cranelift,
where we don't need the cross-engine portability in the benchmark runner, and
where gaming the benchmarks isn't incentivized.

Furthermore, general-purpose WebAssembly benchmarking must include WebAssembly
on the Web. Doing that well requires including interactions with the rest of the
Web browser: JavaScript, rendering, and the DOM. Building and integrating a full
Web browser is overkill for our purposes, and represents significant additional
complexity that we would prefer to avoid.

# Proposal
[proposal]: #proposal

<!-- The meat of the RFC. Explain the proposal in sufficient detail to support -->
<!-- building consensus around the primary design questions and how they affect -->
<!-- stakeholders. The fine details of a design will be finalized during -->
<!-- implementation review. -->

We should design a benchmark suite that contains realistic, representative
programs and diverse programs that collectively represent many different kinds
of program properties and behavior. Running the full benchmark suite should take
a reasonable amount of time. We should take precautions to avoid measurement
bias from uncontrolled variables and other common pitfalls, and the analysis of
the benchmark results should be statistically sound. Finally, the benchmark
suite should have good developer experience: it should be easy to set up and run
on your own machine and reproduce someone else's results.

The result for the Wasmtime and Cranelift community, ultimately, is that we will
have a single, shared, canonical, high-quality benchmark suite. A benchmark
suite where we all have confidence that performance improvements to or
regressions on it are meaningful. We won't be concerned that poor statistical
analysis cloud our results, and we won't dismiss results as "not meaningful"
because the benchmark programs aren't representative.

Making this a reality requires roughly four things:

1. Consensus on what operations to measure and which metrics to record.

2. A corpus of candidate benchmark programs to choose from.

3. A benchmark runner and runtime that helps us avoid measurement bias.

4. Sound statistical analysis of the results.

5. Good developer ergonomics locally and remotely.

## What to Measure and Which Metrics to Record

The life cycle of a WebAssembly module consists of three main operations, and
our benchmark runner should measure all of them:

1. Compilation
2. Instantiation
3. Execution

For each of these operations, there are many different metrics we can record:

* Wall time
* Instructions retired
* Code size
* Cycles
* Cache misses
* Branch misses
* Max RSS
* TLB misses
* Page faults
* Etc...

The benchmark runner should be *capable* of measuring all of these metrics, so
that we can conduct targeted experiments, but *by default* we should focus on
just the three most important metrics:

1. **Wall time:** The actual time it takes for the operation to complete. This
   is the thing we ultimately care about most of the time, so we should measure
   it.

2. **Instructions retired:** Unfortunately, wall time is noisy because so many
   different variables contribute to it. Instructions retired correlates with
   wall time but, by contrast, has significantly less variation across
   runs. This means we can often get a more precise idea of an optimizations
   effect on wall time from fewer samples. This isn't a silver bullet, which is
   why we measure instructions retired in addition to, rather than instead of,
   wall time: the correlation can break down in the face of parallelism or
   instructions are potentially more or less expensive than others (e.g. a
   memory store versus an add).

3. **Max RSS:** The maximum resident set size (amount of virtual memory paged
   into physical memory) for the operation. This gives us coarse-grained insight
   into the operation's memory overhead.

We will automatically report regressions only for these default metrics, and
optimization proposals should always report them. Even if an optimization is
specifically targeting some other metric, it should still report the default
set, and *additionally* report that other metric. Focusing on just these default
metrics avoids information overload and simplifies interpreting results for
engineers. It also reduces the noise of potentially having too many automatic
regression notifications often for "secondary" metrics that aren't significant
in isolation. When engineers generally perceive these warnings as false alarms
then they will stop investigating them, and real, meaningful regressions will
sneak past. Maintaining a healthy culture of performance requires avoiding this
phenomenon!

## Candidate Benchmark Programs

[pca]: https://en.wikipedia.org/wiki/Principal_component_analysis

We intend to collect a corpus of candidate programs for inclusion in the
benchmark suite, quantify these candidates with various static code metrics and
dynamic execution metrics, and finally use [principal component analysis
(PCA)][pca] to choose a diverse subset of candidates that are collectively
representative of the whole corpus. The use of PCA to select representative and
diverse benchmark programs is an established technique, used by both the
DaCapo<sup>[1](#ref-1)</sup> and Renaissance<sup>[2](#ref-2)</sup> benchmark
suites. It has also been used to show that the SPEC CPU benchmarks includes
duplicate workloads whose removal would make SPEC CPU smaller and take less time
to run.<sup>[8](#ref-8)</sup>

The full set of candidates, the metrics instrumentation code, and the PCA
scripts should all be saved in the benchmarks repository. This way we can
periodically add a new batch of candidates, recharacterize our corpus, and
choose a new subset to act as our benchmark suite. We may want to do this, for
example, once interface types are supported in toolchains and Wasmtime, so we
can keep an eye on our interface types performance. The full set of candidates
could also be useful for training PGO builds of Wasmtime and as a corpus from
which we harvest left-hand sides for offline superoptimization.

### Candidate Program Requirements

* Candidates should be real, widely used programs, or at least extracted kernels
  of such programs. These programs are ideally taken from domains where Wasmtime
  and Cranelift are currently used, or domains where they are intended to be a
  good fit (e.g. serverless compute, game plugins, client Web applications,
  server Web applications, audio plugins, etc.).

* A candidate program must be deterministic (modulo Wasm nondeterminism like
  `memory.grow` failure).

* A candidate program must have two associated input workloads: one small and
  one large. The small workload may be used by developers locally to get quick,
  ballpark numbers for whether further investment in an optimization is worth
  it, without waiting for the full, thorough benchmark suite to complete.

* Each workload must have an expected result, so that we can validate executions
  and avoid accepting "fast" but incorrect results.

* Compiling and instantiating the candidate program and then executing its
  workload should take *roughly* one to six seconds total.

  > Napkin math: We want the full benchmark to run in a reasonable amount of
  > time, say twenty to thirty minutes, and we want somewhere around ten to
  > twenty programs altogether in the benchmark suite to balance diversity,
  > simplicity, and time spent in execution versus compilation and
  > instantiation. Additionally, for good statistical analyses, we need *at
  > least* 30 samples (ideally more like 100) from each benchmark program. That
  > leaves an average of about one to six seconds for each benchmark program to
  > compile, instantiate, and execute the workload.

* Inputs should be given through I/O and results reported through I/O. This
  ensures that the compiler cannot optimize the benchmark program away.

* Candidate programs should only import WASI functions. They should not depend
  on any other non-standard imports, hooks, or runtime environment.

* Candidate programs must be open source under a license that allows
  redistributing, modifying and redistributing modified versions. This makes
  distributing the benchmark easy, allows us to rebuild Wasm binaries as new
  versions are released, and lets us do source-level analysis of benchmark
  programs when necessary.

* Repeated executions of a candidate program must yield independent samples
  (ignoring priming Wasmtime's code cache). If the execution times keep taking
  longer and longer, or exhibit harmonics, they are not independent and this can
  invalidate any statistical analyses of the results we perform. We can easily
  check for this property with either [the chi-squared
  test](https://en.wikipedia.org/wiki/Chi-squared_test) or [Fisher's exact
  test](https://en.wikipedia.org/wiki/Fisher%27s_exact_test).

* The corpus of candidates should include programs that use a variety of
  languages, compilers, and toolchains.

### Characterization Metrics

Choosing diverse and representative programs with PCA requires that we have
metrics by which we characterize our candidates. We propose two categories of
characterization metrics:

1. Wasm-level properties (like Wasm instruction mix)

2. Execution-level properties recorded via performance counters

We should write a tool to characterize Wasm programs both by how many of each
different kind of Wasm instruction statically appears in the program, and by how
many are dynamically executed. Measuring these as absolute values would make
larger, longer-executing programs look very different from smaller,
shorter-executing programs that effectively do the same thing. As a trivial
example, a program with all of its loop bodies unrolled N times will look
roughly N times more interesting than the original program without loop
unrolling. Therefore we should normalize these static and dynamic counts. The
static counts should be normalized by the total number of Wasm instructions in
the program, and the dynamic counts normalized by the total number of Wasm
instructions executed.

Proposed buckets of Wasm instructions:

* Integer instructions (e.g. `i32.add`, `i64.popcnt`)
* Float instructions (e.g. `f64.sqrt`, `f32.mul`)
* Control instructions (e.g. `br`, `loop`)
* `call` instructions
* `call_indirect` instructions
* Memory load instructions
* Memory store instructions
* `memory.grow` instructions
* Bulk memory instructions (e.g. `memory.fill`, `memory.copy`)
* Table instructions (e.g. `table.get`, `table.set`, `table.grow`)
* Miscellaneous instructions

This tool should additionally record the Wasm program's initial memory size, its
static maximum memory size, and its dynamic maximum memory size on its given
workload.

The second set of metrics we should record are hardware and software performance
counters (these can be recorded via `perf`, or equivalent):

* Instructions retired
* L1 cache misses (icache and dcache)
* LLC load and store misses
* Branch misses
* TLB misses
* Page faults

Depending on the counter, these can be noisy across runs, so we should take the
mean of multiple samples. Additionally, these counters should all be normalized
by the number of cycles it took to execute the program.

### Initial List of Potential Candidates

Here is an initial brainstorm of potential candidates benchmark programs:

* The [wasmboy](https://github.com/torch2424/wasmboy) gameboy emulator
* The [`rustfmt`](https://github.com/rust-lang-nursery/rustfmt) Rust formatter
* A markdown-to-HTML translator
* Resizing images
* Transcoding video
* [`bzip2`](http://www.bzip.org/) and/or
  [`brotli`](https://github.com/google/brotli) compression
* Some of the [libraries](https://bugzilla.mozilla.org/show_bug.cgi?id=1673529)
  that Firefox is sandboxing with Lucet and Cranelift
* A series of queries against a [SQLite](https://sqlite.org/index.html) database
* The [Twiggy](https://github.com/rustwasm/twiggy) size profiler for WebAssembly
* The
  [`wasm-smith`](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasm-smith)
  WebAssembly test case generator
* The
  [`wasmparser`](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasmparser)
  WebAssembly parser
* The
  [`wat`](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wat)
  WebAssembly assembler
* The
  [`wasmprinter`](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wasmparser)
  WebAssembly disassembler
* The [Clang](https://clang.llvm.org/) C compiler
* A WebAssembly interpreter
* Something to do with [YoWASP](https://yowasp.org/)
* Something to do with [OpenCV](https://opencv.org/)
* Something to do with cryptography
* Some kind of audio filter, processor, decoder, encoder, or synthesizer
* Something to do with regular expressions
* Something to do with protobufs
* Something to do with JSON
* Something to do with XML

This list is in no way complete and we also might not use any of these! Its sole
purpose is only to help kickstart the process of coming up with candidates.

> NOTE: We should *not* block this RFC on coming up with a complete list.
> Finding candidate programs can happen after coming to consensus that we should
> maintain a benchmark suite, agree on a candidate selection methodology, and
> merge this RFC.

## Avoiding Measurement Bias

Seemingly innocuous changes &mdash; the order in which object files are passed
to the linker, the username of the current OS user, the contents of unix
environment variables that aren't ever read by the program &mdash; can result in
significant performance differences.<sup>[3](#ref-3)</sup> Often these
differences come down to the process's address space layout and its affects on
CPU caches and branch predictors. It might seem like a source change sped up a
process, but the actual cause might be that two branch instructions no longer
live at addresses that share an entry in the CPU's branch target buffer. Our
"speed up" is, in other words, accidental and nothing is stopping us from losing
it in the next commit nor is anything guaranteeing that it is visible on another
machine!

An effective approach to mitigating these effects, introduced by the
Stabilizer<sup>[5](#ref-5)</sup> tool, is to repeatedly randomize code and heap
layout while the program is running. This way we avoid getting a single "lucky"
or "unlucky" layout and its associated measurement bias. Stabilizer,
unfortunately, requires both a Clang compiler plugin and a runtime, and hasn't
been updated for the latest LLVM releases. We can, however, take inspiration
from it.

The following measures are not perfect, for example we are missing a way to
randomize static data, and there are almost assuredly "unknown unknowns" that
aren't addressed. Nonetheless, if we implement all of these mitigations, I think
we can have a relatively high degree of confidence in our measurements.

### Randomizing Dynamic Allocation

We should write a "shuffling allocator" wrapper that gives (most) heap
allocations' addresses a uniform distribution. This helps us avoid measurement
bias related to the accidental interaction between allocation order and
caches. The Stabilizer paper shows that this can be done by maintaining a 256
element buffer for each size class. When an allocation is requested make the
underlying `malloc` call, choose a random integer `0 <= i < 256`, swap
`buffer[i]` with the result of the `malloc`, and return the previous
`buffer[i]`. Freeing a heap allocation involves choosing a random integer `0 <=
i < 256`, swapping `buffer[i]` with the pointer we were given to free, and then
freeing the previous `buffer[i]`. Implementing this is straightforward and I
have a prototype already.

For Wasmtime's dynamically mapped JIT code, we should investigate adding a Cargo
feature that randomizes which pages we map for JIT code as well as the order of
JIT'd function bodies placed within those pages.

### Randomizing Stack Layout

We should investigate adding a Cargo feature to Cranelift that adds a random
amount of extra, unused space to each function's stack frame. This padding would
always be a multiple of 16 bytes to keep the stack aligned, and would be at most
4096 bytes in size. This helps us avoid measurement bias related to incidental
stack layout.

### Randomizing Environment Variables

The benchmark runner should define new random environment variables for each
benchmark program execution. This helps us avoid measurement bias related to
incidental address space layout effects from environment variables.

### Randomizing Link Order

We should investigate building a "shuffling linker". Similar to how the
shuffling allocator adds randomness by interposing between `malloc` calls and
the underlying allocator, this would add randomness to the order in which object
files are passed to the linker by interposing between `rustc` and `ld`. Our
shuffling linker will look at the CLI arguments passed to it from `rustc`,
shuffle the order of object files, shuffle object files within `.rlib`s (which
are `.a` files that also contain a metadata file), and finally invoke the
underlying linker. Before running benchmarks, we will create a few different
`wasmtime` binaries that we choose from when sampling different benchmark
executions, each with a different, randomized object file link order. This will
help us avoid measurement bias related to getting "lucky" or "unlucky" with a
single code layout due to the order of object files passed to the linker.

> Note that this approach is more coarsely grained than what Stabilizer
> achieves. Stabilizer randomized code layout at the function granularity, while
> this approach works at the object file granularity.

### Randomizing Benchmark Execution Order

If we are running benchmarks with two different versions of Wasmtime, version A
and version B, we should not take all samples from version A and then all
samples from version B. We should, instead, intersperse executions of version A
and version B. While we should always ensure that the CPU frequency scaling
governor is set to "performance" when running benchmarks, this additionally
helps us avoid some measurement bias from CPU state transitions that aren't
constrained within the duration of process execution, like dynamic CPU
throttling due to overheating.

## Analysis

The worst kind of analysis we can do is something like compare averages with and
without our proposed optimization, and use that to conclude whether the
optimization "works". Or, equivalently, eyeball two bars in a graph and decide
which one looks better. We don't know whether the difference is meaningful or
just noise.

A better approach is to use a test of statistical significance to answer the
question as to whether the difference is meaningful or not. For comparing two
means we would use [Student's
t-test](https://en.wikipedia.org/wiki/Student%27s_t-test) and for comparing
three or more means we would use [analysis of variance
(ANOVA)](https://en.wikipedia.org/wiki/Analysis_of_variance). These tests allow
us to justify statements like "we are 95% confident that change X produces a
meaningful difference in execution time".

We aren't, however, solely interested in whether execution time is different or
not. We want to know how much of a speed up an optimization yields! A
significance test, while an improvement over eyeballing results, is still
"asking the wrong question". This leads us to the best approach for analyzing
performance results: [effect size confidence
intervals](https://en.wikipedia.org/wiki/Effect_size). This analysis justifies
statements like "we are 95% confident that change X produces a 5.5% (+/- 0.8%)
speed up compared to the original system". See the "Challenge of Summarizing
Results" and "Measuring Speedup" sections of *Rigorous Benchmarking in
Reasonable Time*<sup>[4](#ref-4)</sup> for further details.

Our benchmark results analysis code should report effect size along with its
confidence interval. The primary data visualization we produce should be a bar
graph showing speed up or slowdown normalized to the control, with the
confidence intervals overlaid. The R language has an [off-the-shelf
package](https://github.com/easystats/effectsize) for working with effect size
confidence intervals, but we can also always implement this analysis ourselves
should we choose another statistics and plotting language or framework.

## Developer Ergonomics

For optimal developer ergonomics, it should be possible to run the benchmark
suite remotely by leaving a special comment on a github pull request:

> @bytecodealliance-bot bench

This should trigger a github action that runs the benchmark suite on the `main`
branch and the pull request's branch and leaves a comment in the pull request
with the results. This will require dedicated hardware. We will also likely want
to restrict this capability only to users on an explicit allow list.

For greater control over knobs, number of samples taken, which cargo features
are enabled, etc... developers may want to run the benchmark suite
locally. Doing so should be as easy as cloning the repository and executing
`cargo run`:

    $ git clone https://github.com/bytecodealliance/benchmarks
    $ cd benchmarks/
    $ cargo run

The developer could also pass extra CLI arguments to `cargo run` to specify, for
example, the path to a local Wasmtime checkout and a branch name, that only the
small benchmark workloads should be used, or that only a particular benchmark
should be run.

Finally, the benchmark runner should remain usable even in situations where the
user's OS doesn't expose APIs for reading hardware performance counters
(i.e. macOS). It shouldn't be all or nothing; the runner should still be able to
report wall times at minimum.

## Incremental Milestones

What is described above is a significant amount of work. Like all such
engineering efforts, we should gain utility along the way, and shouldn't have to
wait until everything is 100% complete before we receive any return on our
engineering investment. Below is a dependency graph depicting chunks of work and
milestones. Milestones are shown in rectangular boxes, while work chunks are
shown in ellipses. An edge from A to B means that A blocks B.

![](./0000-benchmark-suite-milestones.png)

Here is a short description of each milestone:

* **MVP:** In order to reach an MVP where we are able to get some kind of return
  on our engineering investments, we'll need an initial benchmark runner,
  initial set of candidate programs, and finally a simple analysis that can tell
  us whether a change is statistically significant or not. This is enough to be
  useful for some ad-hoc local experiments. The MVP does *not* mitigate
  measurement bias, it does *not* have a representative and diverse corpus, and
  it does *not* integrate with GitHub Actions or automatically detect
  regressions on the `main` branch.

  Note that the MVP is the only time where we need to implement multiple chunks
  of work before getting any kind of all that we require to start getting any
  returns on our engineering investment. Every additional chunk of work after
  the MVP will immediately start providing benefits as it is implemented, and
  the other milestones only exist to organize and categorize efforts.

* **Measurement Bias Mitigated:** This group of work covers everything required
  to implement our measurement bias mitigations. Once completed, we can be
  confident that the source changes are the cause of any speed up or slow down,
  and not any incidental code, data, or heap layout changes.

* **Excellent Developer Experience:** This group of work covers everything we
  need for an excellent developer experience and integrated workflow. When
  completed we will automatically be notified of any accidental performance
  regressions, and we will be able to test specific PRs if we suspect they might
  have an impact on performance.

* **Representative and Diverse Corpus:** This group of work covers everything
  required to build a representative set of candidate benchmark programs to
  choose from and to select a subset of them for inclusion in the benchmark
  suite. We would choose the subset such that it doesn't contain duplicate
  workloads but still covers the range of workloads represented in the full set
  of candidates. Upon completion we can be sure that our benchmarks make
  efficient use of experiment time, and that speed ups to our benchmark suite
  translate into speed ups in real world applications.

* **Statistically Sound and Rigorous Analysis:** The MVP analysis will only do a
  simple test of significance for the difference in performance with and without
  a given change. This will tell us whether any difference really exists or is
  just noise, but it won't tell us what we really want to know: how much faster
  or slower did it get?! Implementing an effect size confidence interval
  analysis will give us exactly that information.

* **Benchmark Suite Complete:** All done! Once completed, all of our
  benchmarking suite goals will have been met. We can safely kick our feet back
  and relax until we want to update the benchmark candidate pool to include
  programs that use features like interface types.

While doing the MVP first is really the only hard requirement to order of
milestones (and work items within milestones) I recommend focusing efforts in
this order: **MVP** first, **excellent developer experience** second so that
developers get "hooked", followed next by **representative and diverse corpus**
and **statistically sound and rigorous analysis** in any order, and finally
finishing with **measurement bias mitigations**.

# Rationale and Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- What other designs have been considered and what is the rationale for chosing -->
<!-- the one proposed? -->

## Why not reuse an existing benchmark suite?

There are a handful existing WebAssembly benchmark suites:

* [Embenchen](https://kripken.github.io/embenchen/)
* [JetStream2](https://browserbench.org/JetStream2.0/)
* [PSPDFKit WebAssembly
  Benchmark](https://github.com/PSPDFKit-labs/pspdfkit-webassembly-benchmark)
* [`webassembly/benchmarks`](https://github.com/WebAssembly/benchmarks)

These benchmark suites focus on the Web and, even when they don't require a full
Web browser, they assume a JavaScript host. They assume they can import Web or
JavaScript functions and their benchmark runners are written in JavaScript. This
is not suitable for a standalone WebAssembly runtime like Wasmtime, and
depending on these extra moving parts would harm developer ergonomics.

[sightglass]: https://github.com/bytecodealliance/sightglass

The [Sightglass benchmark suite][sightglass] has been used to benchmark Lucet's
(and, by extension, Cranelift's) generated code. It uses the C implementations
of the [Computer Language Benchmarks
Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/) (formerly
the Great Programming Language Shootout) compiled to WebAssembly and then
compiled to native code with Lucet. These programs are
[admittedly](https://benchmarksgame-team.pages.debian.net/benchmarksgame/why-measure-toy-benchmark-programs.html)
toy programs and do not cover the variety of real world programs we intend to
support. It is an open question whether we can or should evolve Sightglass's
runner for this new benchmark suite.

Finally, it is not clear that any of these suites are representative and diverse
since, to the best of my knowledge, none used a statistical approach in their
benchmark program selection methodology.

## Why not contribute to the standard `webassembly/benchmarks` suite?

While the `webassembly/benchmarks` suite currently assumes a Web environment, or
at least a JavaScript host, we could conceivably send pull requests upstream to
make some portion of the benchmarks run in standalone WASI environments. But
this is additional work on our part that ultimately does not bring additional
benefit. Finally, in the roughly year and a half since the project started, only
a single benchmark program has been added to the repository.

> Aside: I'd like to thank Ben Titzer for collecting and sharing the references
> pointed to in `webassembly/benchmarks`. They have been super inspiring and
> helpful!

## Why not compile SPEC CPU to WebAssembly?

[SPEC CPU](https://www.spec.org/cpu/) is a commonly used benchmark suite of C,
C++, and Fortran programs. It is praised for having large, real world
programs. However, it also has a few downsides:

* It is not open source, and distributing the suite is legally questionable. A
  license for SPEC CPU costs $1000.

* It would likely require additional porting efforts to target WebAssembly and
  WASI.

* It takes quite a long time to run the full suite, and research has shown that
  it effectively contains duplicate work loads, meaning that the extra run time
  isn't time well spent.<sup>[8](#ref-8)</sup>

## Why not only measure instructions retired and avoid all this statistics nonsense?

Because wall time is what we ultimately care about. If we only measured
instructions retired, we wouldn't be able to measure potential speed ups from,
for example, adding parallelism.

## Why only support Wasmtime and not Lucet?

The intention is for Lucet to merge into Wasmtime, so we won't need to benchmark
Lucet at all eventually.

# Open Questions
[open-questions]: #open-questions

<!-- - What parts of the design do you expect to resolve through the RFC process -->
<!--   before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through implementation after -->
<!--   the RFC is accepted? -->

* Are there additional metrics we should consider either when characterizing
  candidates when selecting which programs we want to include in the suite or
  when measuring a benchmark program's performance?

* What other candidate benchmark programs should we consider? (Once again: we
  shouldn't block merging this RFC on answering this question, but we should
  agree on selection methodology.)

* Are there additional requirements that candidate benchmark programs should
  meet to be considered for inclusion in the benchmark suite? Anything missing,
  overlooked, or left implicit?

* What should the interface between a candidate benchmark program and the
  benchmark runner be? This opens up a lot of inter-related questions. Should we
  just measure `wasmtime benchmark.wasm` at a process level? If so, how do we
  separate compilation, instantiation, and execution? Do we want to allow
  benchmark programs to set up and tear down state between samples, so that an
  OpenCV candidate, for example, could exclude the I/O required to read its
  model from the measurement? This would imply that the benchmark program either
  doesn't export a `main` function and instead exports `setup`, `tear_down`, and
  `run` functions, or that it imports `"bench" "start"` and `"bench" "end"`
  functions. If we do this, do we want to allow multiple samples from the same
  process? (*Rigorous Benchmarking in Reasonable Time*<sup>[4](#ref-4)</sup> can
  help us decide.)

* Are there additional points we've missed where we should add randomization to
  mitigate another source of potential measurement bias?

* Can, and should, we evolve Sightglass's benchmark runner into what we need for
  this new benchmark suite?

* Should we replace wall time with cycles? This way our measurements should be
  less affected by CPU frequency scaling. On the other hand, wall time is what
  we ultimately care about.

* Should the benchmark runner disable hyper threading? This will read to less
  noisy results, but also a potentially less realistic benchmarking environment.

* Cranelift already has some timing infrastructure to measure which passes
  compile time is spent in. Can we enable this infrastructure when benchmarking
  compile times and integrate its results into our analyses?

* We already noted under the "Developer Ergonomics" section that the benchmark
  runner should remain usable even in environments where it can't record all the
  same metrics it might be able to on a different OS or architecture. But which
  platforms will have first class automatic regression testing and how much work
  will we do to add support for measuring a particular metric across platforms?
  One potential (and likely?) answer is that we will only support reading
  hardware performance counters on x86-64 Linux and will only run automatic
  performance regression testing on x86-64 Linux. All other platform and
  architecture combinations will support measuring wall time for developers to
  run local experiments, but won't have automatic regression testing.

# References
[references]: #references

[renaissance]: https://renaissance.dev/resources/docs/renaissance-suite.pdf
[dacapo]: https://www.cs.utexas.edu/users/speedway/DaCapo/papers/dacapo-cacm-2008.pdf
[rigorous]: http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.881.3551&rep=rep1&type=pdf
[wrong]: https://users.cs.northwestern.edu/~robby/courses/322-2013-spring/mytkowicz-wrong-data.pdf
[stabilizer]: https://people.cs.umass.edu/~emery/pubs/stabilizer-asplos13.pdf
[spec-cpu-search]: https://www.spec.org/cpuv8/
[warmup]: https://arxiv.org/pdf/1602.00602.pdf
[spec-characterization]: https://uweb.engr.arizona.edu/~tosiron/papers/2018/SPEC2017_ISPASS18.pdf

1. <span id="ref-1"/> [*Wake Up and Smell the Coffee: Evaluation Methodology for
   the 21st Century* by Blackburn et al.][dacapo]

   Describes the motivation, design, and benchmark program selection methodology
   of the DaCapo benchmark suite for the JVM. They recorded metrics for each
   candidate benchmark program and then used principal component analysis to
   choose a statistically representative and diverse subset for inclusion in the
   benchmark suite.

2. <span id="ref-2"/> [*Renaissance: Benchmarking Suite for Parallel
   Applications on the JVM* by Prokopec et al.][renaissance]

   Describes the motivation, design, and benchmark program selection methodology
   of the Renaissance benchmark suite for the JVM. Designed to fill gaps that
   were found missing in DaCapo roughly a decade later. They also used principal
   component analysis, but chose more modern characterization metrics with an
   eye towards parallelism and concurrency.

3. <span id="ref-3"/> [*Producing Wrong Data Without Doing Anything Obviously
   Wrong!* by Mytkowicz et al.][wrong]

   Shows that seemingly innocuous, uncontrolled variables (e.g. order of object
   files passed to the link, starting address of the stack, size of unix
   environment variables) can produce large amounts of measurement bias,
   invalidating experiments.

4. <span id="ref-4"/> [*Rigorous Benchmarking in Reasonable Time* by Kalibera et
   al.][rigorous]

   Presents a method to design statistically rigorous experiments that use
   experimental time efficiently. Provides answers to questions like how many
   iterations should I do within a process execution? How many process
   executions should I do with the same binary? How many times should I
   recompile the binary with a different code layout?

5. <span id="ref-5"/> [*Stabilizer: Statistically Sound Performance Evaluation*
   by Curtsinger et al.][stabilizer]

   Presents a runtime and LLVM compiler plugin for avoiding measurement bias
   from fixed address space layouts by randomizing code layout, static data
   placement, and heap allocations.

6. <span id="ref-6"/> [*SPEC CPU v8 Benchmark Search Program* by Standard
   Performance Evaluation Corporation.][spec-cpu-search]

   The detailed, step-by-step SPEC CPU benchmark program submission process, as
   well as an enumeration of the properties they are searching for in a
   candidate, and potential application areas they are interested in.

7. <span id="ref-7"/> [*Virtual Machine Warmup Blows Hot and Cold* by Barrett et
   al.][warmup]

   Questions the common belief that programs exhibit a slower warm up phase,
   where the JIT observes dynamic behavior or shared libraries are loaded and
   initialized, followed by a faster steady state phase. It turns out that many
   programs never reach a steady state! This leads to samples that are not
   statistically independent, invalidating analyses.

8. <span id="ref-8"/> [*A Workload Characterization of the SPEC CPU2017
   Benchmark Suite* by Limaye et al.][spec-characterization]

   Characterizes the instruction mix, exectuoin performance, branch, and cache
   behaviors of the SPEC CPU2017 benchmark suite. Finds that many workloads are
   effectively identical, and suggests ways for researchers to use PCA to choose
   a representative subset of the suite.
