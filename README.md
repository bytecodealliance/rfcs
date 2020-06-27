# Bytecode Alliance RFCs

This respository is the home of the RFC (request for comments) process for Bytecode Alliance projects. RFCs are a tool for getting feedback on design and implementation ideas and for consensus-building among stakeholders.

## What is an RFC?

An RFC is a markdown file laying out a problem and a proposed solution. To support getting feedback early on, RFCs can come in [draft](template-draft.md) or [complete](template-complete.md) forms (see the linked templates for details). Draft RFCs should be opened as [draft PRs](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests#draft-pull-requests). In either case, discussion happens by opening a pull request to place the RFC's markdown file into the `accepted` directory.

## When is an RFC needed?

Many changes to Bytecode Alliance projects can and should happen through every-day GitHub processes: issues and pull requests. An RFC is warranted when:

* The work involves changes that will significantly affect stakeholders or project contributors. Each project may provide more specific guidance. Examples include:
    * Major architectural changes
    * Major new features
    * Simple changes that have significant downstream impact
    * Changes that could affect guarantees or level of support, e.g. removing support for a target platform
    * Changes that could affect mission alignment, e.g. by changing properties of the security model
* The work is substantial and you want to get early feedback on your approach.

## Workflow

### Creating and discussing an RFC

* The RFC process begins by submitting a (possibly draft) pull request, using one of the two templates available in the repository root. The pull request should propose to add a single markdown file into the `accepted` subdirectory, following the template format, and with a descriptive name.

* The pull request is tagged with a **project label** designating the Bytecode Alliance project it targets.

* Once an RFC PR is open, stakeholders and project contributors will discuss it together with the author, raising any points of concern, exploring tradeoffs, and honing the design.

### Making a decision: merge or close

TBD, see [the first RFC](https://github.com/bytecodealliance/rfcs/pull/1).