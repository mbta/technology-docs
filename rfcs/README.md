# RFCs

The RFC process here is inspired by [that of the Rust programming
language](https://github.com/rust-lang/rfcs).

The process is intended to provide TID team members a venue in which to propose
substantial technical changes. The RFC will be reviewed and
merged as either accepted (thereby binding TID engineering teams to the relevant
work), or rejected. The RFCs will remain in the repository and be a source of
historical information for anyone looking back for information on the context
and reasoning that went into some decisions.

This is intended for "larger" changes. As a rough heuristic, the process is for
something that might take a developer an entire sprint to accomplish, or more.
The GitHub interface is pretty good for allowing comments and discussion on
particular points in the RFC, and the RFC can be iterated on with subsequent
commits. After the RFC is merged, the tech lead will turn it into Asana tasks.

We can continue to use Asana for bug reports and smaller tweaks that don't need
larger discussions.

Larger changes that have a user focus might go through a product and design
cycle first. However, for the implementation of the feature, the technical
approach may be opened as an RFC for discussion first.

## When you need to follow this process
We expect that engineers will follow this process when:
- Proposing a technology (language, database, external integration) in use by fewer than 2 other TID teams
- Proposing a change to the integration between applications managed by separate TID teams
- Requested by members of engineering leadership

In other cases, an RFC is not required but is recommended:
- Public facing changes, which might need to be coordinated with other groups inside or outside of TID.
- If you or your peers are unsure about a proposed approach

Changes that don't need an RFC may be:
- A substantial refactor, but entirely internal to an application: this can be an [Architecture Decision Record](https://adr.github.io/)
- Smaller changes, which can use the normal inbound Asana flow.

When in doubt, feel free to ask!

## Before creating an RFC

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, proposals which don't
solve a current problem, or those that don't fit into the near-term roadmap, may
be quickly rejected, which can be demotivating for the unprepared contributor.
Laying some groundwork ahead of the RFC can make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from other developers beforehand, to
ascertain that the RFC may be desirable; having a consistent impact requires
concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking
the idea over in Slack, with the principal architect, or during your 1-on-1 with
your tech lead or engineering leadership. If the proposal is likely to require a
chunk of time to write (for example, if you want to prove out and test a proof
of concept or a part of it), it may be possible to allot sprint time to it.
Submit an inbound Asana task and it will be reviewed and prioritized by the tech
lead and product manager according to the road map and team priorities.

It's worth drafting and getting feedback on an RFC even before formally
proposing it. A fresh set of eyes may be able to quickly identify and resolve
some issues before it's shared with the wider organization, making the
discussion of the formal proposal that much more productive.

## What the process is

- Copy `rfcs/0000-template.md` to `rfcs/0000-my-feature.md` (where "my-feature" is descriptive).
  Don't assign an RFC number yet; This is going to be the PR number and we'll rename the file
  accordingly once the RFC is opened.
- Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation,
  demonstrate lack of understanding of the design's impact, or are disingenuous about the drawbacks
  or alternatives tend to be poorly-received.
- Submit a pull request. As a pull request the RFC will receive design feedback from the team, and
  the author should be prepared to revise it in response.
- Now that your RFC has an open pull request, use the issue number of the PR to update your 0000-
  prefix to that number.
- Announce it in the [`#tid-engineering`](https://mbta.slack.com/archives/GP5QJM98A) Slack channel for all the team to review.
- RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown.
  You can make edits, big and small, to the RFC to clarify or change the design, but make changes as
  new commits to the pull request, and leave a comment on the pull request explaining your changes.
  Specifically, do not squash or rebase commits after they are visible on the pull request.
- At some point, one of the RFC editors will propose a "motion for final comment
  period" (FCP), along with a disposition for the RFC (accept or reject).
  - This step is taken when enough of the tradeoffs have been discussed that the
    editor is in a position to make a decision. That does not require consensus
    amongst all participants in the RFC thread (which is usually impossible).
    However, the argument supporting the disposition on the RFC needs to have
    already been clearly articulated, and there should not be a strong consensus
    against that position. The editor will use their best judgment in taking
    this step, and the FCP itself ensures there is ample time and notification
    for stakeholders to push back if it is made prematurely.
  - For RFCs with lengthy discussion, the motion to FCP is usually preceded by a
    summary comment trying to lay out the current state of the discussion and
    major tradeoffs/points of disagreement.
- When the RFC is finalized:
  - The RFC author moves the document to the appropriate folder (`accepted` or `rejected`).
  - One of editors approves the PR.
  - The RFC author merges the PR.

## The RFC life-cycle

Once an RFC becomes "active" then the RFC becomes binding on TID teams. The RFC
author is responsible for creating at least one Asana task for each team
impacted by the RFC, summarizing the expected change for the team. That
associated task can be assigned a priority via the triage process that the team
uses for all work. The fact that a given RFC has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a developer has been assigned the task of
implementing the feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We strive
to write each RFC in a manner that it will reflect the final design of the
feature; but the nature of the process means that we cannot expect every merged
RFC to actually reflect what the end result will be.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes should
be new RFCs, with a note added to the original RFC. Exactly what counts as a
"very minor change" is up to the editor to decide.

## Editors

The editors who can approve an RFC are:

- Ryan Mahoney (Director of Enginering)
- Ian Westcott (Director of Infrastructure)
- Matt Shanley (Deputy Director of Engineering)
- Paul Swartz (Principal Architect)

## Reviewing RFCs

While the RFC pull request is up, the editor may schedule meetings with the author and/or
relevant stakeholders to discuss the issues in greater detail, in which case a summary 
from the meeting will be posted back to the RFC pull request. Reviewers will be looking
to see that the outcome of the RFC would meet the goals set out in the 
[Motivation](0000-template.md#motivation) and that there is sufficient detail in the other
sections to address any relevant questions or risks.

## A "living" process

The RFC process itself can be modified, and that's encouraged! To do so, simply submit a PR with
changes to this README or to the template document.
