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
the API and not in our GTFS feed. The API will gain a new "occupancies" resource accessible via
joining to trips. The data will be documented in the API's swagger docs as "Experimental" with a
warning that it could change in the future, and require a new `x-enable-experimental-features`
header. The implementation will be as minimal and simple as possible, but nevertheless built in a
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
whole trip or individual stop times along the trip. A given occupancy specifies its `status` which
is an enum ranging from `0` for "Empty" to `6` for "Full". Similar to how we represent vehicles in
the API, the API response will be the enum word rather than the number. Our feed will have an
_extra_ field, currently not part of the spec, but which is being advocated for, `percentage`,
which is a value from `0` to `100`. Each occupancy has some validity constraints, namely the start
and end dates for which they're valid. A trip could therefore have several related occupancies.
Different from the spec we're not including stop sequence or valid days of the week yet.

Our Keolis source data is at the trip level, and that is the only producer for which we will have
static occupancy data in the near term. Initially, we will put this data directly in the API and
not in our GTFS feed since the data updates as frequently as once/hour, and we don't want to
update our GTFS static feed that often. However, we should ensure the API interface works _as if_
it were coming from the GTFS feed, in case we do so in the future. While we're not given this
data, we can arbitrarily say that the occupancies are valid for two weeks.

Our API largely matches the GTFS feed and is simply a convenient way of accessing that data. While
there are a few GTFS resources that we use in the API that are not exposed directly (e.g.
transfers, calendar) and one major one that's named differently (schedule vs stoptime), the
default expectation is that GTFS resources are available in the API.

To that end, the API is able to include occupancies on trips. Since the data model will allow for
multiple occupancies per trip, if they vary along the way per stop, each trip will get a list of
occupancies. For now, though, with the Keolis data, that list will have just a single element.

For now, due to the experimental and malleable nature of the proposal, we want to start small and
only allow including occupancies on trips. While it's natural to extend the inclusion to schedules
and other resources, we will start with the minimum necessary for the current use case. For
simplicity we don't even need a `/occupancies` as its _own_ endpoint at all.

```
GET /trips?filter[route]=CR-Worcester&include=occupancies

{
  data: [
    {
      "attributes": {
        "bikes_allowed": 1,
        "block_id": "",
        "direction_id": 1,
        "headsign": "South Station",
        "name": "500",
        "wheelchair_accessible": 1
      },
      "id": "CR-483657-500",
      "links": { "self": "/trips/CR-483657-500" },
      "relationships": {
        "route": { "data": { "id": "CR-Worcester", "type": "route" } },
        "route_pattern": { "data": { "id": "CR-Worcester-c7319c64-1", "type": "route_pattern" } },
        "service": { "data": { "id": "SummerWeekday", "type": "service" } },
        "shape": { "data": { "id": "9850001", "type": "shape" } },
        "occupancies": {
          "data": [
            { "id": "occupancy-CR-483657-500", "type": "occupancy" },
          ]
        }
      },
      "type": "trip"
    }
    // ...
  ],
  included: [
    {
      "attributes": {
        "status": "MANY_SEATS_AVAILABLE",
        "percentage": 11,
        "start_date": [today],
        "end_date": [2 weeks from today],
      }
      "id": "occupancy-CR-483657-500",
      "type": "occupancy"
    }
  ]
}
```

This should be enough for dotcom to do the CR timetable pages. In particular, since it's based off
the `/trips` endpoint, they will be able to filter by route and direction, which might be useful.

In order for the `include=occupancies` to work, a client will have to use the special header
`x-enable-experimental-features: true`.

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
  :status,
  :percentage,
  :start_date,
  :end_date
]
```

- The `:trip_id` can be looked up using a trimmed version of `cTrainNo` which is equivalent to a
  trip's GTFS `trip_short_name`.
- `:status` can in theory be one of `:empty`, `:many_seats_available`, `:few_seats_available`,
  `:standing_room_only`, `:crushed_standing_room_only`, `:full`, or `:not_accepting_passengers`,
  but for the Keolis data will only use 3 of them, with the mapping of `MedianDensityFlag`s `0`,
  `1`, and `2` to the GTFS values that dotcom uses in real-time crowding for 1, 2, and 3 bowling
  pins respectively: `:many_seats_available`, `:few_seats_available`, and `:full`.
- `:percentage` is the `MedianDensity` multiplied by 100 and rounded.
- `:start_date` is the current date
- `:end_date` is the date two weeks from today

The index will need to be able to allow efficient selection by `trip_id`.

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

## API /trips updates

Fill out the swagger docs for the new `include` option, but make sure it says **EXPERIMENTAL** and
"can change without prior warning". The docs should state that to opt-in to experimental features
the request must pass a `x-enable-experimental-features: true` header.

The `ApiWeb.TripView` will need to be updated to load the occupancies for a trip. It should be
specified as `has_many` even though it will only have one (for now).

## Experimental feature

Add a new plug that looks for `x-enable-experimental-features: true` and adds
`:experimental_features_enabled?` to the `conn`'s assigns. The trip controller should consider
that value when validating the request's includes.

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

The main reason to avoid this right now is the data changes too frequently, on the order of
once/hour, while we don't usually publish a feed more than once or twice a day. Plus the frequent
changes would make it harder for clients (who aren't using `occupancies.txt` yet) to know if the
data they care about has changed.

These issues could be worked around. For example:

- We could have a frequently updating feed and a more significant snapshot feed.
- We could publish `occupancies.txt` separately from the main zip file.
- We could just update the static data less frequently than the source data changes.

But for now, in order to avoid setting that other stuff up, and to take full advantage of the
Keolis data, we'll have the API poll the firebase feed directly, and in the future (maybe when
there's data from other modes, too?), evaluate new ways for the API to get the data.

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

## Via an /occupancies endpoint

Rather than attaching to trips, we could have provided an `/occupancies` endpoint. This also works
but is likely a larger development effort.

It also pushes a lot of the uncertainty around the proposal to the client. For example, should we
have occupancies be 1-to-1 or 1-to-many for each trip? Implemented via `/occupancies` the API
implementor does not need to decide. But the client, in interpreting the results, for example
grouping by trip ID, will need to decide. And if they see that there is only one occupancy per
trip, they might rely on that, in which case adding multiple occupancies per trip in the future at
the data level, is still something of a breaking change. By the API making the decision now and
asserting it as 1-to-many despite the current state of the data, we lock ourselves into that
position, but it's more robust for the client.

By tying the implementation to trips, we also get the API's extensive filtering functionality,
e.g. by route or direction or both, or date, etc, for free.

# Prior art

[prior-art]: #prior-art

The IBM parking feed is a custom parser which is part of the API.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

None anymore.

# Future possibilities

[future-possibilities]: #future-possibilities

We will likely evolve this feature according to the GTFS proposal as it develops, and perhaps as
we have access to more data.

We could introduce other modes' data. We already calculate subway historical crowding, so it's
plausible we might add this. One consideration is that there are `ADDED-` subway trips. These
don't map neatly to `occupancies.txt`, unless we want to use its `start_time` field in conjunction
with `frequencies.txt`.

Ultimately, we'd like to integrate occupancies into the API more in the future. The most likely
extension would be to allow them to be included on queries of schedules and predictions. The details
would need to be worked out since the spec doesn't explicitly associate an occupancy with a stop time, since
there could be multiple occupancies for a given stop time, with different valid ranges or days of
week. We might wish to support something like:

```
GET /schedules?filter[xyz]=abc&include=occupancies

{
  "data": [
    {
      "id": "schedule-[trip_id]-[time]",
      "attributes": { ... },
      "relationships": {
        "occupancies": {
          "data": [
            {"id": "some_occupancy", "type": "occupancy"},
            ...
          ]
        }
      }
    }
  ],
  "included": [
    {
      "id": "some_occupancy",
      "attributes": { ... }
    },
    ...
  ]
}
```

Note that the schedule's relationship to `"occupancies"` is a list. This would include all the
occupancies that are valid at different times for that stop time.

We might have a similar `include` available for `/predictions`, which works similarly.

Or, we might want to allow or require filtering by dates to change the occupancy to
1-to-1, rather than 1-to-many, as this might be easier for a client to work with.

I'm not sure if there's precedent in the API but we could try to distinguish `include=occupancies`
which would return all vs `include=occupancy` which tries to attach the "best" or only valid
occupancy to a given schedule.
