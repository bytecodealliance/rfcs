# WIT-Bindgen Versioning and Release Process

# Summary
As more users come to rely on `wit-bindgen` the need for a thorough versioning policy and release process grows. We should address these needs as soon as possible, aiming to provide clarity for our users and communicate changes that will affect them while still allowing the project to grow and change.

# Motivation
[motivation]: #motivation

* **Communicate changes to users.** Users that depend on `wit-bindgen`'s various crates need to know when things change so that they can keep their documentation and code up to date.

* **Provide guarantees to users.** Users that depend on `wit-bindgen` need to be able to rely on typical assumptions made about versioned software. Specifically, that our published and versioned artifacts comply with [Semantic Versioning](https://semver.org/).

* **Maintain development flexibility / agility** Since `wit-bindgen` is not finalized and will need to adapt as bindgen use cases and the Component Model evolve and mature, the release process and versioning need to make it possible for us to continue making changes.

# Proposal sketch
[proposal]: #proposal

## Contents of Wit-Bindgen

The `wit-bindgen` repository contains a few different kinds of crates that may be treated differently.

* **wit-bindgen CLI** - The binary crate that can be used as a Command Line Interface (CLI) for generating bindings using any of the Language Bindings Generators.
* **Common Utilities** - Any facilities shared by individual Language Bindings Generators including the WIT Interface Definition Language parser.
* **Language Bindings Generators** - The various crates that each provide support for generating bindings for a singular scenario.
* **Rust Bindings Wrappers** - Wrapper crates around the host and guest Language Bindings Generators in the language being generated so that they can be used within that languages workflow. There is currently only support for this in Rust.

## Definitions
* A **Release PR** is a PR where the release of new versions of `wit-bindgen` crates is proposed.
* A **Release Commit** is the commit created by the completion of the *Release PR* and signals that the release has officially occurred.
* The **Release Lead** is the person responsible for the next release.
* An **updated** crate is one which has had source changes since the last *Release Commit*.
* A **released** crate is one which has a new version introduced by this *Release Commit*.
* A **skipped** crate is one which does not have a new version introduced by this *Release Commit*.

## Versioning

### WIT
This versioning and release process will not define the way the WIT Interface Definition Language is updated and versioned.
Crates that rely on WIT must increment their versions as necessary when updated to match changes to WIT.
* If WIT becomes separate and versioned, crates must identify the version of WIT they support.
* Unless/until then, crates must support the version of WIT as described in the repo itself.

### Crates
All crates are treated as their own independently versioned artifacts.
Crates will be published to [crates.io](https://crates.io/) as needed by users.

* The *Common Utilities* crates will follow [Semantic Versioning 2.0](https://semver.org/).
* The *Language Bindings Generators* and *Rust Bindings Wrappers* crates will follow [Semantic Versioning 2.0](https://semver.org/). Both the semantics of WIT they understand and the characteristics of the code they generate will be considered part of their public API and taken into account when versioning them.
* The `wit-bindgen` CLI crate will **NOT** be Semantically Versioned, because it would need to increment its major version anytime any of the individual *Language Bindings Generators* do, which would cause it to rise rapidly.
  * If the semantics of WIT that the `wit-bindgen` CLI supports change, a breaking change will increment its major version and a non-breaking change will increment its minor version.
  * If a *Language Bindings Generator* crate has a breaking change, only the `wit-bindgen` CLI's minor version will be updated and a non-breaking change will increment its patch number.

As a result of this versioning policy, users should depend on the version of the specific *Language Bindings Generator* they use if possible.

## Contribution Process
All contribution Pull Requests titles and descriptions must conform to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) and not use the type `release`.

## Release Process
A new release begins when the current *Release Lead* creates a *Release PR*.

* The *Release PR* must conform to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) and have a title formatted as `release: <released-crates>`,
  * where `released-crates` is a comma-separated list of the crates names of all *released* crates.
* The changelog of each *updated* crate must be updated.
  * A *released* crate’s changelog must contain a new section for the new version containing all previously unreleased changes and any changes since the last release event.
  * A *skipped* crate’s changelog must contain an "Unreleased" section containing all previously unreleased changes and any changes since the last release event.
* The version number of each *released* crates must be set to a valid subsequent version according to our [versioning policy](#versioning).
* The *Release PR* description must designate the next *Release Lead*.
* The discussion of a *Release PR* may be used to debate, discuss, and/or revise the content it proposes releasing.
* When the *Release PR* is merged to master, the *released* crates it identifies are considered approved, and the title becomes the commit message of the *Release Commit*.

When the *Release PR* has been merged to master, the *Release Lead* will trigger the release pipeline.
It will determine the crates to release based on the `released-crates` list in the *Release Commit* title.

## Tooling
This release process has been designed so that tooling can be configured or made to automate as much of it as possible.
The intention is that the *Release Lead* will be able to create the *Release PR* by running a command line utility that will
 1. traverse the commit history back to the last *Release Commit*,
 2. determine what kind of changes have occurred to each artifact,
 3. update the changelogs based on the Conventional Commits format commit titles,
 4. propose a new version number for each *updated* crate,
 5. prompt the user to choose which crates to mark as *released*


and then render all of these changes to the source files and then indicate what the *Release PR* title should be to release the selected crates.

Ideally, this tooling will be smart enough to identify when a crates folder path changes automatically based on the crate names in `Cargo.toml` files. Handling the names of crates changing is complicated and those changes should probably be made manually after running the tool so that the history traversal never encounters a rename.

# Open questions
[open-questions]: #open-questions

* Will the `wit-bindgen` repository ever include non-Rust artifacts that need to be released?
* How do we effectively communicate the versioning policy of the `wit-bindgen` CLI to users?
* How do we encourage and enable users to depend on the individual language bindings generator they are using?