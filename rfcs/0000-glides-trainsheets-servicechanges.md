- Feature Name: `glides-trainsheets-servicechanges`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)
- Asana task: [asana link](https://app.asana.com/)
- Status: Proposed

# Summary
[summary]: #summary

| GTFS-realtime | Realtime Edits |
|---------------|----------------|
|Glides will provide trainsheet-based service data to RTR and other services that need it by providing a GTFS-ServiceChanges JSON feed via AWS S3.|Glides will provide trainsheet-based service data to RTR and other services that need it by sending [CloudEvents](https://cloudevents.io/)-formatted data to applications that wish to consume that data.|

# Motivation
[motivation]: #motivation

As part of digitizing the currently paper-based trainsheets that define operational conditions on the Green Line, Glides will be gathering information about trip assignments and adjustments from its users.
This data can serve to improve the rider-facing predictions produced by RTR, especially for departure times at a terminal, which currently do not have departure predictions at all.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Glides trainsheets can record several different types of events.

<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Events are added to the Glides trainsheet data feed as soon as they are entered to Glides, and are removed from the feed five minutes after the relevant trip has ended or was scheduled to end, whichever comes last.

Example top-level feed structure:
```json
{
  "header": {
    "timestamp": 1649102174,
  },
  "entity": [
    // ... events as described below
  ]
}
```

</td>
<td>

Events are sent to consumers as soon as they are entered to Glides.

</td>
</tr>
</table>

## Trip Assignments

Glides trainsheets record the consist that will be assigned to each trip.
Currently, RTR has to guess this assignment based on limited information.

When Glides knows the consist assigned to a trip that already exists in static GTFS,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

it uses a GTFS-realtime VehicleUpdate event to convey that information in the feed:
```json
{
  "id": "01dd55e08e36",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50974586"
    },
    "vehicle": {
      "label": "3856-3846",
      "consist": [
        {
          "label": "3856"
        },
        {
          "label": "3846"
        }
      ]
    }
  }
}
```

This way of handling the consist matches what's currently in the public GTFS-realtime VehiclePositions enhanced JSON feed.

</td>
<td>

it sends a trip assignment event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.trip-assignment",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50974586",
    "consist": [
      "3856",
      "3846"
    ]
  }
}
```

</td>
</tr>
</table>

## Dropped Trips

If a trip listed in static GTFS will not be occurring at all,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides uses a GTFS-realtime VehicleUpdate event to mark the trip as canceled:
```json
{
  "id": "ac378f6daace",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50974586",
      "schedule_relationship": "CANCELED"
    }
  }
}
```

</td>
<td>

Glides sends a dropped trip event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.dropped-trip",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50974586"
  }
}
```

</td>
</tr>
</table>

## Added Trips

If a trip that will be occurring cannot be matched to a trip listed in static GTFS,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides uses a GTFS-realtime VehicleUpdate event to mark the trip as a duplicate of an existing trip running the same path:
```json
{
  "id": "b9a1e9112231",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50974586", // from static GTFS
      "schedule_relationship": "DUPLICATED"
    },
    "trip_properties": {
      "trip_id": "757460c22ece", // new
      "start_date": "20220404",
      "start_time": "15:57:09"
    }
  }
}
```

</td>
<td>

Glides sends an added trip event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.added-trip",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "template-trip-id": "50974586",
    "new-trip-id": "757460c22ece",
    "service-date": "2022-04-12",
    "start-time": "15:57:09"
  }
}
```

</td>
</tr>
</table>

## Adjusted Trips - Skipped Stops

If a trip will be skipping some stops it was scheduled to make, either because it is express or because its destination has been brought earlier in its path,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides uses a GTFS-realtime VehicleUpdate event to mark the applicable stops as skipped:
```json
{
  "id": "f9b7373a3fc4",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50974586"
    },
    "stop_time_update": [
      {
        "stop_id": "70129",
        "stop_sequence": 230,
        "schedule_relationship": "SKIPPED"
      }
    ]
  }
}
```

</td>
<td>

Glides sends an adjusted trip event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.adjusted-trip",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50974586",
    "skipped-stops": [
      "70129"
    ]
  }
}
```

</td>
</tr>
</table>

## Adjusted Trips - Added Stops

If a trip will be making more stops than it was scheduled to make,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides uses a [GTFS-ServiceChanges v3.1](https://bit.ly/gtfs-service-changes-v3_1) NewTrips event to provide the new sequence of stops and a GTFS-ServiceChanges v3.1 VehicleUpdate event to mark the original trip as replaced by the new trip:
```json
{
  "id": "84e68f49dfa5",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50973989",
      "schedule_relationship": "CANCELED",
      "replaced_by_trip_id": "43c2bea52540"
    }
  },
  "trip": {
    "trip_id": "43c2bea52540",
    "trip_headsign": "Kenmore",
    "direction_id": 1,
    "stop_time": [
      {
        "stop_sequence": 17,
        "arrival_time": "24:36:00",
        "departure_time": "24:36:00",
        "stop_id": "70504"
      },
      {
        "stop_sequence": 18,
        "arrival_time": "24:40:00",
        "departure_time": "24:40:00",
        "stop_id": "70502"
      },
      {
        "stop_sequence": 19,
        "arrival_time": "24:42:00",
        "departure_time": "24:42:00",
        "stop_id": "70208"
      },
      {
        "stop_sequence": 20,
        "arrival_time": "24:45:00",
        "departure_time": "24:45:00",
        "stop_id": "70206"
      },
      {
        "stop_sequence": 30,
        "arrival_time": "24:46:00",
        "departure_time": "24:46:00",
        "stop_id": "70204"
      },
      {
        "stop_sequence": 40,
        "arrival_time": "24:48:00",
        "departure_time": "24:48:00",
        "stop_id": "70202"
      },
      {
        "stop_sequence": 80,
        "arrival_time": "24:50:00",
        "departure_time": "24:50:00",
        "stop_id": "70199"
      },
      {
        "stop_sequence": 90,
        "arrival_time": "24:52:00",
        "departure_time": "24:52:00",
        "stop_id": "70159"
      },
      {
        "stop_sequence": 100,
        "arrival_time": "24:54:00",
        "departure_time": "24:54:00",
        "stop_id": "70157"
      },
      {
        "stop_sequence": 110,
        "arrival_time": "24:55:00",
        "departure_time": "24:55:00",
        "stop_id": "70155"
      },
      {
        "stop_sequence": 120,
        "arrival_time": "24:57:00",
        "departure_time": "24:57:00",
        "stop_id": "70153"
      },
      {
        "stop_sequence": 130,
        "arrival_time": "24:59:00",
        "departure_time": "24:59:00",
        "stop_id": "71151"
      }
    ]
  }
}
```

If there was no original trip, the `trip_update` value makes no reference to an original trip:
```json
{
  "trip_update": {
    "trip": {
      "trip_id": "43c2bea52540",
      "schedule_relationship": "ADDED"
    }
  }
}
```

</td>
<td>

Glides sends an adjusted trip event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.adjusted-trip",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50973989",
    "replaced-stops": [
      "70504",
      "70502",
      "70208",
      "70206",
      "70204",
      "70202",
      "70199",
      "70159",
      "70157",
      "70155",
      "70153",
      "71151"
    ]
  }
}
```

</td>
</tr>
</table>

## Departure Time

If the time at which a trip will leave its starting terminal is entered into Glides,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides will use a GTFS-realtime TripUpdate event to specify that time:
```json
{
  "id": "2131d0f347c1",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50973989"
    },
    "stop_time_update": [
      {
        "stop_id": "70504",
        "stop_sequence": 17,
        "schedule_relationship": "SCHEDULED",
        "arrival": {
          "time": 1649104744
        },
        "departure": {
          "time": 1649104744
        }
      }
    ]
  }
}
```

</td>
<td>

Glides sends a start time event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.start-time",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50973989",
    "start-time": "16:39:04"
  }
}
```

</td>
</tr>
</table>

## Hold Time

If Glides knows that a train will be held for extra time at a station,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides will use a GTFS-realtime TripUpdate event to specify that delay:
```json
{
  "id": "17b4cd8574a3",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50973989"
    },
    "stop_time_update": [
      {
        "stop_id": "70204",
        "stop_sequence": 30,
        "schedule_relationship": "SCHEDULED",
        "departure": {
          "delay": 180 // seconds
        }
      }
    ]
  }
}
```

</td>
<td>

Glides sends a hold time event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.hold-time",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50973989",
    "stop-id": "70204",
    "hold-time": 180 // seconds
  }
}
```

</td>
</tr>
</table>

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

## Alternative: Glides pushes new information to RTR

## Alternative: Glides pushes new information to an AWS Kinesis stream

## Alternative: GTFS-ServiceChanges over protobuf

## Alternative: GTFS-realtime + internal extensions

## Alternative: internal JSON

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what
this can include are:

- Can we learn something about this proposal from other projects in CTD?
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
