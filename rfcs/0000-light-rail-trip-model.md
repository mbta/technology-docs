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

The TID data architecture does not exist in a vacuum. For our purposes in this RFC, the main outside factor that we need to be aware of is the HASTUS system used for scheduling and the Plans & Schedules team that uses it to produce the new light rail schedule every rating. Challenges in this area, in particular around schedules during disruptions, will be factored in.

Finally, I want to take a second to lay out some guiding principles and goals that have guided the decisions in this RFC:

1. Having a single source of truth for data.
1. Creating clear understanding between teams about ownership of different parts of the data pipeline.
1. Balancing the needs of internal operations users and public-facing users.
1. Staying as close as possible to the most basic common denominator in data across TID systems, in this case GTFS trips. This means that improvements benefit as many teams and applications as possible, and we adhere to the principle of "small pieces, loosely joined" by a common standard.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Schedule data: HASTUS and TODS

Glides will move towards using the HASTUS-provided trip ID in its schedule data.

(Update with any future thoughts based on TODS conversations.)

(Also to go here: high-level overview of how Glides will handle these changes, including things like DB schema updates, if any)

## Realtime data: Glides trainsheets and RTR

In the first stage, RTR and Glides will continue to maintain their own separate trip assignment systems, but Glides will begin sending HASTUS-based trip IDs in trainsheet update events. For scheduled trips, the [trip key published in `glides.trips_updated` events](https://mbta.github.io/schemas/events/glides/com.mbta.ctd.glides.trips_updated.v1#tripkey) will need to be updated in a new version of the schema to include this ID. The `serviceDate` will still need to be present in order to map from HASTUS schedule IDs to GTFS service IDs.

Glides will still need to accommodate added trips. When RTR receives an added trip from Glides, it should factor the given trip ID in for vehicle assignments. However, if a train that is assigned to an added trip from Glides changes state in such a way that it should be assigned to a different trip by RTR's logic (for instance: unexpectedly going off its current pattern, or changing directions), RTR is free to assign that train to a different trip of its choosing. RTR will still be free to create its own added trips as needed.

In a future state, Glides will be responsible for light rail trip assignments, and generate a new event feed of train-to-trip assignment events.

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

In deciding which system would be the source of truth for assignments between trains and trips, I had to weight the pros and cons of both RTR and Glides taking that logic on, ultimately landing on having it live in Glides. The following factors influenced my decision:

1. Assigning trips to trains is a fundamentally stateful exercise that depends on a trains progress on its previous trip, location, trainsheet data about upcoming trips, and other factors. RTR has some stateful logic, but its state storage and management system is not very advanced. Glides has a relational database that is already updated with information about train movements and trips as they happen.
1. This mirrors the situation on heavy rail where the OCS performs realtime trip assignments and RTR uses those as its main source of input for HR trip assignments.

The main downsides of putting this logic in Glides are:

1. As Glides will be more directly responsible for data that flows more directly into passenger-facing systems, it is possible that the Glides team will have to respond to more feedback about issues related to why Green Line trains were assigned to incorrect trips. However, this is likely going to happen anyway with trainsheet-based predictions in RTR given that that already relies on input from Glides.
1. Glides does not currently have a way of getting location data on yard pull-out / pull-back trains from RTR, so we will need to set up a pipeline for that in order for Glides to implement the full functionality. However, we already need to do this to enable other functionality in Glides, so it is not introducing net new work.
1. Glides will need to generate automatically-created trips for trains in service that aren't matched to any trip on a trainsheet, logic that is currently implemented in RTR. However, this is likely desirable anyway because Glides will need a place to associate things like trip notes on unmatched trains in the future.
1. Glides may not be aware of some disruptions in its schedule data that are modeled at the `gtfs_creator` level, meaning that we will still need RTR to create ADDED trips for any scheduled trips that it doesn't recognize. With the current logic, RTR will at least attempt to match to trips in the disrupted GTFS. However, given that these trips aren't guaranteed to match actual operational practice, it's unclear how much value there is in matching to them anyway.
1. It introduces a very slight lag in trip assignments making their way into passenger-facing data when updated due to train movements. RTR will need to receive the updated location information and Glides will need to receive it, and then Glides will in turn need to make the assignment and transit that information to RTR via an event. However, this added delay is likely minimal, especially if we move towards an event-based architecture for sending vehicle position updates from RTR to Glides (out of scope for this RFC but also something I want to consider in the future).

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

For the purposes of this RFC, we have had to assume that there will still be planned disruptions that have not been modeled in HASTUS in a way that reflects actual operations and trainsheets. This leaves open questions as to whether this will always be the case, and whether or not we will one day be able to assume that the schedule from HASTUS reflects planned operations in all cases.

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
