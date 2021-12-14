- Feature Name: gtfs_canonical_routes
- Start Date: 2021-06-03
- RFC PR: [mbta/technology-docs#0006](https://github.com/mbta/technology-docs/pull/0006)
- Asana task: [🚝 Publish canonical trips/patterns for rail routes](https://app.asana.com/0/881264583703207/1200325279789524/f)
- Status: Proposed

# Summary

This will enable us to publish in our static GTFS feed "canonical" data for our rail routes. We will
do so via a "canonical" service that's never active, but which has representative trips with stop
times and shapes. If a canonical trip is similar enough to a real-world trip that are otherwise
in the GTFS feed, then it will be made the representative trip of an existing route pattern. If no
such trips are being run (such as due to a long-running station closure), then a new, canonical
route pattern will be created, of which the canonical trip will be representative.

# Motivation

MBTA.com users are able to see line diagrams that match real-world state, including physical maps,
even when stations are closed for a long duration.

PIOs are able to create alerts for stations without any scheduled service, and T-Alerts subscribers
are able to stay informed about the status of the station closure.

# Guide-level explanation

GTFS output includes additional, hardcoded data about where rail lines go to and where they can stop
if all stations were open; these hardcoded data are independent of all scheduled service except that
they may be tied to scheduled route patterns in some cases.

This is represented as a service with ID "canonical", which is valid for the entire feed date range,
but which has no active days of the week. The typicality of the service is tagged with a new value
so it can be programmatically identified.

All rail routes have two or more trips on this service, corresponding to the "canonical" patterns of
service in each direction for any branches, as understood by stakeholders. Each such canonical trip
has a route pattern, for which it is the representative trip, and a shape. If current service has
trips following the same set and order of stops as the canonical trip, then the canonical trip will
reference—and become the representative trip for—the current service's route pattern. If current
service has no trips resembling the canonical trip, then a separate, canonical route pattern will be
maintained in the GTFS output.

# Reference-level explanation

The canonical service appears in the following GTFS files in the following way:

- **calendar.txt** includes a new `service_id` to envelop all of the canonical trips on all the
  routes. `start_date` and `end_date` should reflect the feed's valid start/end dates. All other
  fields should have a value of `0`.
- **calendar_attributes.txt** includes the new `service_id`. `service_schedule_type` should be
  `"Other"`, and `rating_start_date` and `rating_end_date` should reflect the feed's valid start/end
  dates. `service_schedule_typicality` should be `6`, and the
  [docs](https://github.com/mbta/gtfs-documentation/blob/master/reference/gtfs.md) should be updated
  with this new kind of value.
- **route_patterns.txt** includes new route patterns for routes/directions we want to hardcode when
  the canonical trips (added in the trips file below) do not resemble any scheduled service. New
  patterns in this file shall have a new `route_pattern_typicality` value of should be `6`, and
  `route_pattern_sort_order` should be a greater than the value for all non-canonical route patterns
  for that given route/direction. This will be done by adding a new digit to
  `route_pattern_sort_order` (placed right after the existing digits corresponding to the
  `route_sort_order`). For routes with branches, you'll need a pattern for each
  route/direction/branch (eg, for Providence/Stoughton Line, you should have _two_ new route
  patterns in the outbound direction). Finallly, a new `canonical_route_pattern` field is added to
  all route patterns, indicating whether or not it can be considered canonical (or if the route does
  not have canonical patterns defined, as will be the case for bus initially).
- **shapes.txt** will add new shapes for all canonical subway trips, since the shape IDs can change
  between ratings, especially when stations are open and closed on a long-term basis. These new
  shapes can have an identical set of points as those shapes used during current service. Existing
  Commuter Rail shape references can be reused for canonical trips.
- **trips.txt** includes one new trip for each canonical route/direction pair, or multiple trips in
  the event of regular branching occuring along the route. It should reference the new `service_id`
  created above. It should reference either a new (canonical) or previously-existing (regular
  service) route pattern, depending on whether the trip's stopping pattern is currently served by
  regular service. Each of these trips should be the `representative_trip_id` for their referenced
  route patterns.
- **stop_times.txt** includes all stops that trains in the route/direction passes through, even if
  they are temporarily closed. Increment the arrival/departure times by one minute for each
  subsequent stop. For a sample, see the attached GTFS file, the output of [✓ 🧪 🚝 Canonical
  trips/patterns experiment](https://app.asana.com/0/881264583703207/1200210504369250), which
  trialled this for Green Line D and Lowell Line (only in the outbound direction).

This will be done as a new "step" of the Makefile called `canonical`, which runs between `truncate`
and `current`. There will be a new a directory, `input/canonical/` to store any static data the step
uses.

The canonical step will be a Python module that reads in the `truncate` feed, and writes out its own
feed after transforming it. The transformations are as above, and also will update any canonical
trips to use current route patterns (and vice versa) if relevant, eliminating redundant route
patterns.

For Commuter Rail route patterns, the canonical one will specify every stop as non-flag, non-leave
early stop. For both subway and Commuter Rail canonical trips, no data around bicycle allowances
will be provided (`bikes_allowed=0`).

# Drawbacks

This is more complicated than not doing it.

# Rationale and alternatives

See proposed Concepts [1](https://github.com/mbta/technology-docs/pull/6#issuecomment-952315958), [4](https://github.com/mbta/technology-docs/pull/6#issuecomment-952315958), and [5](https://github.com/mbta/technology-docs/pull/6#issuecomment-962120434) in prior discussion.

# Prior art

Until now we've been doing ad-hoc adjustments to determine the route pattern of routes
[here](https://github.com/mbta/api/blob/master/apps/state/config/config.exs#L145).

# Unresolved questions

Ensuring that we would not be in a situation where the sort order, perhaps ordered by
frequency/proportion of trips, becomes a different order than desired for the main route pattern
against the branch route pattern.

How route patterns containing canonical representative trips are tagged or otherwise identified:
Using a new `canonical_route_pattern` field in route_patterns.txt, or is the new
`service_schedule_typicality` field sufficient?

# Future possibilities

We could expose this more clearly in the API. In the approach outlined here, the "canonical" route
pattern is simply that with a typicality of `1`. In the alternative approach, the typicality would
be `5`. That's not currently filterable in the API. In addition, the `service` can't be filtered by
its typicality, most relevantly when included with `trips`.

Should we do something similar for ferry and/or bus?
