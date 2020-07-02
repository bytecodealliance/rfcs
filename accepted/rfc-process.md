# RFC process for the Bytecode Alliance

# Summary

As the Bytecode Alliance (BA) grows, we need more formalized ways of communicating about and reaching consensus on major changes to core projects. This document proposes to adapt ideas from Rust’s [RFC](https://github.com/rust-lang/rfcs/) and [MCP](https://forge.rust-lang.org/compiler/mcp.html) processes to the Bytecode Alliance context.

# Motivation

There are two primary motivations for creating an RFC process for the Bytecode Alliance:

*   **Coordination with stakeholders**. Core BA projects have a growing set of stakeholders building on top of the projects, but not necessarily closely involved in day-to-day development. This group will only grow as the BA brings on more members. An RFC process makes it easier to communicate _possible major changes_ that stakeholders may care about, and gives them a chance to weigh in.

*   **Coordination within a project**. As the BA grows, we hope and expect that projects will be actively developed by multiple organizations, rather than just by a “home” organization. While day-to-day activity can be handled through issues, pull requests, and regular meetings, having a dedicated RFC venue makes it easier to separate out discussions with far-ranging consequences that all project developers may have an interest in.

# Proposal

The design of this RFC process draws ideas from Rust’s [Request for Comment](https://github.com/rust-lang/rfcs/) (RFC) and [Major Change Proposal](https://forge.rust-lang.org/compiler/mcp.html) (MCP) processes, adapting to the BA and trying to keep things lightweight.

## Stakeholders

Each core BA project has a formal set of **stakeholders**. These are individuals, organized into groups by project and/or member organization. Stakeholders can block proposals, and conversely having explicit sign-off from at least one individual within each stakeholder group is a sufficient (but not necessary) condition for immediately accepting a proposal. Stakeholders are not necessarily members of the BA.

The process for determining core BA projects and their stakeholder set will ultimately be defined by the Technical Steering Committee, once it is in place. Until then, the current BA Steering Committee will be responsible for creating a provisional stakeholder arrangement, as well as deciding whether to accept this RFC.

## Structure and workflow

### Creating and discussing an RFC

*   We have a dedicated bytecodealliance/rfcs repo that houses _all_ RFCs for core BA projects, much like Rust’s rfcs repo that is shared between all Rust teams. 
*   The rfcs repo will be structured similarly to the one in Rust:
    *   A template markdown file laying out the format of RFCs, like [Rust’s](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) but simplified.
    *   A subdirectory holding the text of all accepted RFCs, like [Rust’s](https://github.com/rust-lang/rfcs/tree/master/text).
*   New RFCs are submitted via pull request, and both technical and process discussion happens via the comment thread.
*   The RFC is tagged with **project labels**, corresponding to the BA project(s) it targets. This tagging informs tooling about the relevant stakeholder set.

### Making a decision: merge or close

*   When discussion has stabilized around the main points of contention, any stakeholder can make a **motion to finalize**, using a special comment syntax understood by tooling. This motion comes with a disposition: merge or close.
*   In response to the motion to finalize, a bot will post a special comment with a **stakeholder checklist**. 
    *   This list includes the GitHub handle for each individual stakeholder, organized into stakeholder groups. 
    *   The individual who filed the motion to finalize is automatically checked off.
*   Once _any_ stakeholder from a _different_ group has signed off, the RFC will move into a 10 day **final comment period** (FCP), long enough to ensure that other stakeholders have at least a full business week to respond.
*   During FCP, any stakeholder can raise an **objection** using a syntax understood by the bot. Doing so aborts FCP and labels the RFC as `blocked-by-stakeholder` until the objection is formally resolved (again using a special comment syntax).
*   Finally, the RFC is automatically merged/close if either:
    *   The FCP elapses without any objections.
    *   A stakeholder from _each_ group has signed off, short-cutting the waiting period.

## What goes into an RFC?

RFCs will follow a format inspired by Rust, but significantly simplified. RFCs are markdown files containing the following sections (which will be laid out in a template file):

*   **Summary**. A ~one paragraph overview of the RFC.
*   **Motivation**. What problem does the RFC solve?
*   **Proposal**. The meat of the RFC.
*   **Rationale and alternatives**. A discussion of tradeoffs: why was the proposal chosen, rather than alternatives?
*   **Open questions**. Often an RFC is initially created with a broad proposal but some gaps that need community input to fill in.

### Draft RFCs

It is encouraged to use the RFC process to discuss ideas early in the design phase, before a _full_ proposal is ready. Such RFCs should be marked as [a draft PR](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests#draft-pull-requests), and contain the following sections:

*   **Motivation**. What problem are you ultimately hoping to solve?
*   **Proposal sketch**. At least a short sketch of a possible approach. Beginning discussion without _any_ proposal tends to be unproductive.
*   **Open questions**. This section is especially important for draft RFCs: it’s where you highlight the key issues you are hoping that discussion will address.

RFCs cannot be merged in draft form. Before any motion to merge, the draft should be revised to include all required RFC sections, and the PR should be changed to a standard GitHub PR.

## What requires an RFC?

When should you open an RFC, rather than just writing code and opening a traditional PR?

*   When the work involves changes that will significantly affect stakeholders or project contributors. Each project may provide more specific guidance. Examples include:
    *   Major architectural changes
    *   Major new features
    *   Simple changes that have significant downstream impact
    *   Changes that could affect guarantees or level of support, e.g. removing support for a target platform
    *   Changes that could affect mission alignment, e.g. by changing properties of the security model
*   When the work is substantial and you want to get early feedback on your approach.

# Rationale and alternatives

## Stakeholder and FCP approach

This proposal tries to strike a good balance between being able to move quickly, and making sure stakeholder consent is represented.

The core thinking is that getting active signoff from at least one person from a different stakeholder group, together with the original motion to finalize, is sufficient evidence that an RFC has received external vetting. The additional 10 day period gives all stakeholders the opportunity to review the proposal; if there are any concerns, or even just a desire to take more time to review, any stakeholder can file an objection. And on the other hand, proposals can move even more quickly with signoff from each group.	

In total, this setup allows proposals to move more quickly than either Rust’s RFC or MCP processes, due to the less stringent review requirements and ability to end FCP with enough signoff. At the same time, the default FCP waiting period and objection facility should provide stakeholders with enough notice and tools to slow things down when needed.

## Organizations vs individuals

Motion through the FCP process is gated at the stakeholder group level, but the actual check-off process is tied to individuals. There are reasons for both:

*   **Gated by group**. The goal of the signoff process is to provide a “stakeholder consent” check before finalizing an RFC. We view a signoff from _any_ member of an organization or project as a fair representation of that organization or projects’s interests, and expect individuals to act accordingly.
*   **Individual sign-off**. Despite being gated by groups, we call out individuals by GitHub handle for review. Doing so helps avoid diffusion of responsibility: if we instead merely had a checkbox per organization, it could easily create a situation where no particular individual felt “on duty” to review. In addition, tracking which individual approved the review provides an important process record in case that individual fails to represent their group's interests.

## Comparison to Rust’s RFC and MCP processes

The Rust RFC process has successfully governed the development of the language from before 1.0, and covers an enormous range of subprojects, including the language design, tooling, the compiler, documentation, and even the Rust web site. It is a proven model and variations of the process have been adopted by several other large projects. Its workflow is simple and fits entirely within GitHub, which is already the central point of coordination within the BA. And given the central role of Rust within the BA, it’s a model that members are likely to already be familiar with.

That said, a major difference in design is the notion of **stakeholders** and how they impact the decision-making process. Here we borrow some thinking from Rust’s lightweight MCP process, allowing a decision to go forward after getting buy-in from just one stakeholder within a different group -- but still requiring a waiting period to do so. A further innovation is that getting sign off from within _all_ groups immediately concludes FCP. That was not possible in the Rust community, where the set of stakeholders is unbounded.

The most common complaint about the Rust RFC process is that it is, in some ways, a victim of its own success: RFC comment threads can quickly become overwhelming. While this may eventually become an issue for the BA as well, we have some additional recourse: we have a clear membership model which will allow us to prioritize concerns from member organizations, and take action when individuals outside of any organization overwhelm comment threads.

We could instead follow Rust’s MCP model and use Zulip streams for RFC discussion. However, that introduces a more complex workflow (spanning multiple systems) and leaves a less permanent and less accessible record of discussion.

# Open questions

None at this time.