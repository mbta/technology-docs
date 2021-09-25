- Feature Name: api_cr_crowding
- Start Date: 2021-09-23
- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)
- Asana task: [asana link](https://app.asana.com/)
- Status: Proposed

# Summary

[summary]: #summary

We want to provide predicted commuter rail crowding data for upcoming trips. The data will be
consumed from a feed provided by Keolis. We will follow the lead of this [static GTFS-Occupanices
proposal](https://github.com/google/transit/pull/240), but for now the data will only appear in
the API and not in our GTFS feed. The API will gain a new "occupancies" resource. The data will be
documented in the API's swagger docs as "Experimental" with a warning that it could change in the
future. The implementation will be as minimal and simple as possible, but nevertheless built in a
solid and (hopefully) forward-compatible way. The immediate concrete use case for this feature is
showing crowding information on dotcom's CR timetables pages.

# Motivation

[motivation]: #motivation

This data feed will allow clients to easily see expected crowding for CR trips. In particular,
dotcom is planning to add this information to the CR timetable pages.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

<!-- Explain the proposal as if it was already implemented and you were teaching it to a new developer
that just joined the team. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how programmers should _think_ about the feature, and how it should impact the way they
  work on this project. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to senior developers and to junior
  developers.
-->

This introduces into the API the proposed new GTFS static resource called
[occupancies.txt](https://github.com/google/transit/pull/240/files), which contains information
about "usual or expected in-vehicle occupancy levels". The underlying data can apply to either a
whole trip or individual stop times along the trip. A given occupancy specifies its
`occupancy_status` which is an enum ranging from `0` for "Empty" to `6` for "Full". Similar to how
we represent vehicles in the API, the API response will be the enum word rather than the number.
Our feed will have an _extra_ field, currently not part of the spec, but which is being advocated
for, `occupancy_percentage`, which is a value from `0.0` to `1.0`. Each occupancy has some
validity constraints, namely the start and end dates and days of the week for which they're valid.
A trip could therefore have several related occupancies.

Our Keolis source data is at the trip level, and that is the only producer for which we will have
static occupancy data in the near term. Initially, we will put this data directly in the API and
not in our GTFS feed since the data updates as frequently as once/hour, and we don't want to
update our GTFS static feed that often. However, we should ensure the API interface works _as if_
it were coming from the GTFS feed, in case we do so in the future. While we're not given this
data, we can arbitrarily say that the occupancies are valid for two weeks and apply to all days of
the week. A weekend train trip generally doesn't run on weekdays, but if it were to (e.g. Labor
Day running Sunday service), we'd want the predicted occupancy to be considered valid.

Our API largely matches the GTFS feed and is simply a convenient way of accessing that data. While
there are a few GTFS resources that we use in the API that are not exposed directly (e.g.
transfers, calendar) and one major one that's named differently (schedule vs stoptime), the
default expectation is that GTFS resources are available in the API.

To that end, the API has a new `/occupancies` endpoint for an `Occupancy` resource. While the
occupancy must have an ID – which we'll construct as `occupancy-[trip-id]` – there's no
`/occupancies/[id]` endpoint, akin to `/predictions` or `/schedules`.

For now, due to the experimental and malleable nature of the proposal, we want to start small and
limit the coupling of occupancies with the rest of our code. So at a minimum we will provide just
a single `/occupancies` endpoint that returns all current occupancies, with their trip
relationship, like such:

```
GET /occupancies

{
  data: [
    {
      "attributes": {
        "occupancy_status": MANY_SEATS_AVAILABLE,
        "occupancy_percentage": 0.11,
        "start_date": [today],
        "end_date": [2 weeks from today],
        "monday": 1,
        "tuesday": 1,
        "wednesday": 1,
        "thursday": 1,
        "friday": 1,
        "saturday": 1,
        "sunday": 1
      }
      "id": occupancy-[trip-id]-all,
      "relationships": {
        "trip": {
          "data": {
            "id": trip_id,
            "type": "trip"
          }
        },
      },
      "type": "occupancy"
    },
    // ...
  ]
}
```

From there we will add the minimum extra functionality required for our one known consumer,
dotcom. This _might_ be enough as-is. I could foresee an implementation where they periodically
query this endpoint, store it indexed by trip ID and that's enough for the timetable page.

However, I could imagine a couple extensions they might need:

- `?filter[route]=CR-route1`, `?filter[trip]=trip1,trip2,trip3`, or
  `?filter[trip.name]=name1,name2,name3`. I'm not sure how the timetables work, but maybe querying
  only relevant occupancies via route, trip, or trip short name is necessary.
- `?include=trip`. Maybe filtering isn't necessary, but being able to include the trip, in order
  to have easy access to the trip short name, is necessary.

But for the time being, we won't allow other resources to include occupancies. For example, we
don't want clients to do something like `/trips?filter[route]=CR-Worcester&include=occupancies`.
In the event that something substantial changes about occupancies (or if it's removed!), the
changes are limited to just this one resource, which is a risk of using the experimental resource.
On the other hand, if this proposal matures and we build forth, clients who have built
interactions with this endpoint will continue to have this as a supported method of access.

# Reference-level explanation

<!--[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work. -->

## Keolis data

The Keolis data comes from a firebase feed. It looks like:

```
{
  "data": [
    {
      "cTrainNo": "003     ",
      "MedianDensity": 0.58635,
      "MedianDensityFlag": 1
    },
    //...
  ],
  "last_updated": "2021-09-24 15:30:00"
}
```

Each record in `"data"` is for one trip, with `"cTrainNo"` being the train's name (with extraneous
padding for arcane reasons on Keolis's end).

The `"MedianDensity"` is the percentage of seats occupied, and `"MedianDensityFlag` is an enum
with values `0`, `1`, or `2`, representing Keolis's idea of "Not Crowded", "Some Crowding", and
"Crowded" respectively.

## API data

In the API, we'll add a new kind of model `Occupancy` with its own `State.Server`. The model will
have keys (following the GTFS proposal):

```
[
  :trip_id,
  :stop_sequence,
  :occupancy_status,
  :occupancy_percentage,
  :valid_days,
  :start_date,
  :end_date,
  :start_time
]
```

- The `:trip_id` can be looked up using a trimmed version of `cTrainNo` which is equivalent to a
  trip's GTFS `trip_short_name`.
- `:stop_sequence` will be `nil`
- `:occupancy_status` can in theory be one of `:empty`, `:many_seats_available`,
  `:few_seats_available`, `:standing_room_only`, `:crushed_standing_room_only`, `:full`, or
  `:not_accepting_passengers`, but for the Keolis data will only use 3 of them, with the mapping
  based on `MedianDensityFlag` of `0` to `:many_seats_available`, `1` to `:few_seats_available`,
  and `2` to `:full` (unless dotcom already has a different mapping of GTFS terms to bowling
  pins?).
- `:occupancy_percentage` is just the `MedianDensity` value verbatim.
- `:valid_days` is the list `[1, 2, 3, 4, 5, 6, 7]` representing this data applying to all days of
  the week (akin to how `Service` works in the API).
- `:start_date` is the current date
- `:end_date` is the date two weeks from today
- `:start_time` is `nil`.

The index (/indices) will depend on what sort of filtering (if any) we provide, based on above.

## Fetching the data

The Keolis endpoint should be fetched every ten minutes. We could use with modifications the
existing `StateMediator` infrastructure to do this. State mediators can be configured to fetch
from a URL on a periodic basis, but they take `url` as a string configuration, while the URL for
the firebase feed changes periodically because it has a `?token=....` parameter, and that token is
only valid for a short while. So `StateMediator.Mediator` should be updated to optionally take a
MFA tuple for its URL configuration as well.

From futzing around in a Livebook, some code to download the data from Keolis is:

```ex
Mix.install([:jason, :hackney, goth: "1.3.0-rc.3"])
firebase_url = "..."
credentials_json = "..."
credentials = Jason.decode!(credentials_json)

scopes = [
  "https://www.googleapis.com/auth/firebase.database",
  "https://www.googleapis.com/auth/userinfo.email"
]

source = {:service_account, creds, scopes: scopes}
{:ok, pid} = Goth.start_link(name: MyGoth, source: source)
{:ok, token} = Goth.fetch(MyGoth)
resp = :hackney.request(firebase_url <> ".json?access_token=" <> token.token)
{:ok, 200, _, ref} = resp
{:ok, data} = :hackney.body(ref)
Jason.decode!(data)
```

The `Mediator` can invoke `new_state` on its configured module (`State.Occupancy`), which will
then do the mapping and merging of the data as specified above.

## API controller

Fill out the swagger docs, but make sure it says **EXPERIMENTAL** and "can change without prior
warning".

The `includes` and `filters` sections depend on what is decided above, about what is necessary for
dotcom.

# Drawbacks

[drawbacks]: #drawbacks

A drawback of building on GTFS-Occupancies at this point, is that it's still in flux and can
change in the future. But we think that it's better to build on what's there, which will hopefully
be close to the final product (it's gone through lots of revisions at this point and seems to be
settling down) rather than do our own thing and then end up with a totally different way of
handling this data in the future.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## Actual GTFS

The proposal we are modeling after is a static GTFS file `occupancies.txt`. An approach that we
could have taken is for `gtfs_creator` to use the Keolis endpoint in its build process, and then
the API to download the data via the GTFS feed.

The main reason to avoid this is the data changes too frequently, on the order of once/hour, while
we don't usually publish a feed more than once or twice a day. Plus the frequent changes would
make it harder for clients (who aren't using `occupancies.txt` yet) to know if the data they care
about has changed.

## Small, temporary endpoint

We could have done a hacky temporary `/temp_commuter_rail_crowding` endpoint. This would be low
effort, and is appropriately quarantined from the rest of the code.

However, this would set an unhappy precedent about "temporary" endpoints just for dotcom that
don't need to coexist with the rest of the app. This flies in the face of the API's philosophy to
this point, of being an open source app that could, in principle, be used by any transit agency,
and that adheres to transit standards for the most part.

And the most likely outcome of this "experiment" is that it lingers indefinitely. We don't want a
"temporary", ad-hoc endpoint perpetually in the API, especially when its functionality overlaps
with a GTFS standard that will (eventually) be accepted, and we'll want to adopt formally, for
Transit app, Google Maps, etc.

## Not exposed directly, but joined to other models

Another approach is to not have an `/occupancies` endpoint but still have the model and data,
either simply stuck in a `Trip`, or `include=occupancies` by a trip.

The API has a precedent for something like this, where it digests `calendar.txt` but doesn't
expose a `Service` resource or endpoint.

Adding the given `occupancy_status` and `occupancy_percentage` values right into the trip, would
be a straightforward approach. However, this is contrary to the data model in the current
proposal, which explicitly rejects adding these as columns to `trips.txt` or `stop_times.txt`. The
reasoning given is that a given trip could have different occupancy predictions for different date
spans or days of the week. While we could handle this in the API via giving trips a _list_ of
occupancies, this isn't feasible in `trips.txt` in GTFS. And so in the name of forward
compatibility with the proposal once it lands, we shouldn't do it that way now.

The GTFS-compatible way of attaching a list of occupancies to a trip in the API would be to still
have the `Occupancy` resource, and allow `trips` to `include=occupancies`. I'd be open to this
idea, if dotcom thought it would be the most natural way to work with the data, though I'm
hesitant about it because I'm not exactly sure of the joining logic here, given the data model
issues mentioned before. Should we validate date range and days of the week as part of the join?
My gut feeling is that an isolated `/occupancies` endpoint would be simpler for us to build and
maintain while the proposal is being worked on. Having to work with occupancies via trips also
seems like it would couple client implementations between their standard trips and this
non-standard feature more.

# Prior art

[prior-art]: #prior-art

n/a

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- What filters and includes does dotcom absolutely need on the `/occupancies` endpoint, to do
  their work? Can they get by with a simple, unfiltered `/occupancies` with the data and the trip
  ID in it?
- Would dotcom strongly prefer retrieving `occupancies` via an `include` on a trip instead?
- Does dotcom already have an established mapping of GTFS crowding terms to bowling pins? We
  should be in line with that in our translation of Keolis's "not crowded", "some crowding",
  "crowded" flags to GTFS terms.

# Future possibilities

[future-possibilities]: #future-possibilities

The future possibilities are further integration and fleshing out of this feature, once the GTFS
proposal lands. Probably at that point we would introduce it into our GTFS feed, and update the
API to pull the data as part of the GTFS import instead.
