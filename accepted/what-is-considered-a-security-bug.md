# Summary
[summary]: #summary

<!-- One paragraph explanation of the proposal. -->

Wasmtime has documented processes for reporting, responding to, patching, and
disclosing security vulnerabilities. However, the Wasmtime project does not
currently define which kinds of bugs are and are not considered security
vulnerabilities. This RFC aims to reach consensus on that definition.

# Motivation
[motivation]: #motivation

<!-- Why are we doing this? What use cases does it support? What is the expected -->
<!-- outcome? -->

Without a definition of what is and is not considered a security vulnerability,
two different groups of people can reasonably make different assumptions about
whether a particular bug is a security vulnerability. And both groups could be
right in some sense.

For example, if Wasmtime goes into an infinite loop when compiling Wasm modules
of a certain shape, the Wasmtime developers might not consider that a security
vulnerability. After all, it can't lead to an escape from the Wasm
sandbox. However, an attacker might be able to leverage this into a denial of
service against a particular Wasmtime embedding, and the embedders might
therefore reasonably consider these hangs in Wasmtime a security vulnerability.

These differences in assumptions can create friction between the two groups and
damage working relationships. We would like to avoid that and foster healthy
collaboration between all stakeholders in the Wasmtime project.

Additionally, knowing what is not considered a security vulnerability by the
Wasmtime developers, but could nevertheless potentially be leveraged by
attackers, helps Wasmtime embedders design and architect their applications
appropriately. Resuming the above example, if the Wasmtime developers chose not
to consider Wasm compilation that fails to terminate as a security vulnerability
and advertised that fact, then the Wasmtime embedders might architect their
application to compile Wasm in a separate OS process. This way they can apply a
timeout to compilation and simply kill the process if it hangs.

# Proposal
[proposal]: #proposal

<!-- The meat of the RFC. Explain the proposal in sufficient detail to support -->
<!-- building consensus around the primary design questions and how they affect -->
<!-- stakeholders. The fine details of a design will be finalized during -->
<!-- implementation review. -->

We propose (1) defining what the Wasmtime and Cranelift projects consider and
not consider to be security bugs, and (2) documenting this in a new page in [the
Wasmtime book][book].

[book]: https://docs.wasmtime.dev

## Definition

The security of the host and integrity of the sandbox when executing Wasm is
paramount. Anything that undermines the Wasm execution sandbox is a security
vulnerability.

On the other hand, execution that diverges from Wasm semantics (such as
computing incorrect values) are not considered security vulnerabilities so long
as they remain confined within the sandbox. This has a couple repercussions that
are worth highlighting:

* Even though it is safe from the *host's* point of view, an incorrectly
  computed value could lead to classic memory unsafety bugs from the *Wasm
  guest's* point of view, such as corruption of its `malloc`'s free list or
  reading past the end of a source-level array.

* Wasmtime embedders should never blindly trust values from the guest &mdash; no
  matter how trusted the guest program is, even if it was written by the
  embedders themselves &mdash; and should always validate these values before
  performing unsafe operations on behalf of the guest.

Denials of service when *executing* Wasm (either originating inside compiled
Wasm code or Wasmtime's runtime subroutines) are considered security
vulnerabilities. For example, if you configure Wasmtime to run Wasm guests with
the async
[fuel](https://docs.rs/wasmtime/latest/wasmtime/struct.Config.html#method.consume_fuel)
mechanism, and then executing the Wasm goes into an infinite loop that never
yields, that is considered a security vulnerability.

Denials of service when *compiling* Wasm, however, are not considered security
vulnerabilities. For example, an infinite loop during register allocation is not
a security vulnerability.

Any kind of memory unsafety (e.g. use-after-free bugs, out-of-bounds memory
accesses, etc...) in the host is always a security vulnerability.

### Cheat Sheet: Is this bug considered a security vulnerability?

| Type of bug                                     | At Wasm Compile Time | At Wasm Execution Time |
|-------------------------------------------------------------------------------------|-----|-----|
| Sandbox escape                                                                      | -   | Yes |
| <ul>Uncaught out-of-bounds memory access                                            | -   | Yes |
| <ul>Uncaught out-of-bounds table access                                             | -   | Yes |
| <ul>Failure to uphold Wasm's control-flow integrity                                 | -   | Yes |
| <ul>File system access outside of the WASI file system's mapped directories         | -   | Yes |
| <ul>Use of a WASI resource without having been given the associated WASI capability | -   | Yes |
| <ul>Etc...                                                                          | -   | Yes |
| Divergence from Wasm semantics (without escaping the sandbox)                       | -   | No  |
| <ul>Computing incorrect value                                                       | -   | No  |
| <ul>Raising errant trap                                                             | -   | No  |
| <ul>Etc...                                                                          | -   | No  |
| Memory unsafety                                                                     | Yes | Yes |
| <ul>Use-after-free                                                                  | Yes | Yes |
| <ul>Out-of-bounds memory access                                                     | Yes | Yes |
| <ul>Use of uninitialized memory                                                     | Yes | Yes |
| <ul>Etc...                                                                          | Yes | Yes |
| Denial of service                                                                   | No  | Yes |
| <ul>Panic                                                                           | No  | Yes |
| <ul>Process abort                                                                   | No  | Yes |
| <ul>Uninterruptible infinite loops                                                  | No  | Yes |
| <ul>User-controlled memory exhaustion                                               | No  | Yes |
| <ul>Uncontrolled recursion over user-supplied input                                 | No  | Yes |
| <ul>Etc...                                                                          | No  | Yes |

**If you read this cheat sheet and are still unsure whether an issue you are
filing is a security vulnerability or not, always err on the side of caution and
report it as a security vulnerability!**

Note that every bug presented above is still desirable to fix even if it is not
a security vulnerability! We appreciate when issues are filed for
non-vulnerability bugs, particularly when they come with test cases and steps to
reproduce!

## New Page in the Wasmtime Book

The previous definition and cheat sheet will be copied to a new page titled
"What is Considered a Security Vulnerability?" in [the Wasmtime
book][book]. This new page will be a child of [the existing "Security"
page](https://docs.wasmtime.dev/security.html).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- What other designs have been considered and what is the rationale for chosing -->
<!-- the one proposed? -->

* We could consider denials of service when compiling Wasm a security
  vulnerability. This could work for Winch when it matures, but won't work well
  for Cranelift. Cranelift compilation isn't architected for interruption and
  adding it after the fact would be quite a large undertaking and would probably
  introduce new performance overhead.

* On the flip side, we could consider all kinds of denials of service as not
  security bugs, even when they are within Wasm execution. However, unlike our
  main compiler, our runtime is already architected to be resilient in this way,
  and we might as well take advantage of it.

* We could consider all Wasm semantics divergences as security bugs even if they
  remain confined to within the sandbox, for example computing incorrect
  values. This has historically been our position, but this RFC is proposing
  downgrading these bugs. Experience has shown that this causes undue alarm
  among Wasmtime embedders who see a new CVE and drop everything, but then it is
  ultimately just a relatively inconsequential bug. We don't want to get into a
  "boy who cried wolf" kind of relationship with our embedders where people stop
  taking our CVEs seriously and then don't pay attention by the time there is a
  highly critical vulnerability that needs to be patched.

# Open questions
[open-questions]: #open-questions

<!-- - What parts of the design do you expect to resolve through the RFC process -->
<!--   before this gets merged? -->

<!-- - What parts of the design do you expect to resolve through implementation after -->
<!--   the RFC is accepted? -->

* Are we missing anything from our definition of what is a security
  vulnerability?

* Are there other kinds of bugs that we don't consider security vulnerabilities
  but should explicitly mention that?
