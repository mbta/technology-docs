- Feature Name: Consistent Light Rail Trip Model Across TID Applications
- Start Date: 2024-05-16
- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)
- Asana task: [asana link](https://app.asana.com/)
- Status: Proposed

# Summary
[summary]: #summary

This document establishes a North Star vision to guide us towards a shared model of light rail trips that supports various TID goals, primarily in the Transit Technology program area but also having relevance to Rider Tools.

# Motivation
[motivation]: #motivation

The concept of a "trip" is core to essentially all transit systems, and therefore to all transit data as well. Trips are associated with, and tie together, many other important entities in our data model, including:

- Route
- Vehicle
- Operator(s)
- Headsign
- Block
- Run
- Various service adjustments or interventions
- Scheduled stop times
- Predicted stop times
- Actual stop times (in the past for historical instances of trips)

This list is not exhaustive, and in some applications these other entities or data points may be accessed by joining via other parts of the data model as opposed to looking directly at a trip, but hopefully it impresses the importance getting trips right.

Currently, however, our model of trip data on light rail leaves a great deal to be desired. For historical reasons, MBTA light rail vehicles have had neither on-vehicle systems by which an operator can assign their vehicle to a particular run (as in the case with bus) or a centralized dispatch system with its own model of trips and trip assignment (as with heavy rail). Furthermore, inconsistencies in how light rail trips are modeled across systems, related to peculiarities like having both a lead car and a trailer car that both need to have an operator scheduled, lead to more difficulty in exchanging data across systems. This situation creates a variety of tangible problems:

1. Glides needs to perform complicated logic when importing schedule data from HASTUS to transform it into a model that matches the Glides trainsheet interface.
1. Our passenger information (RTR) and dispatch (Glides) applications both implement their own logic to associate vehicles with trips, duplicating effort across teams while also creating additional confusion.
1. Glides does not publish trainsheet changes using trip IDs that directly correspond to the trip IDs used by GTFS, further complicating the logic in RTR that has to assign trips to vehicles.
1. Passenger-facing systems such as mbta.com or screens that may wish to display both scheduled and predicted departure information can't always be sure if two trips with different IDs in the schedule versus the realtime data are actually "the same."
1. Analysis of historical data for performance purposes is also complicated by the difficulty in linking a trip from the schedule with a trip from the (historical) realtime data.

In addition to the technical challenges, this also creates unclear ownership of different parts of the data pipeline, with different teams implementing their own logic piecemeal to serve their own needs.

The TID data architecture does not exist in a vacuum. For our purposes in this RFC, the main outside factor that we need to be aware of is the HASTUS system used for scheduling and the Plans & Schedules team that uses it to produce the new light rail schedule every rating.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already implemented and you were teaching it to a new developer
that just joined the team. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how programmers should *think* about the feature, and how it should impact the way they
  work on this project. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to senior developers and to junior
  developers.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what
this can include are:

- Can we learn something about this proposal from other projects in TID?
- Can we learn something about this proposal from other transit agencies?
- Can we learn something about this proposal from past experiences in other jobs or projects?

This section is intended to encourage you as an author to think about the lessons from other places,
and provide readers of your RFC with a fuller picture.

If there is no prior art, that is fine.

Note that while precedent is some motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would
affect the project as a whole in a holistic way. Try to use this section as a tool to more fully
consider all possible interactions with the project in your proposal. Also consider how this all
fits into the roadmap for the project.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but
otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot
think of anything.

Note that having something written down in the future-possibilities section is not a reason to
accept the current or a future RFC; such notes should be in the section on motivation or rationale
in this or subsequent RFCs. The section merely provides additional information.
