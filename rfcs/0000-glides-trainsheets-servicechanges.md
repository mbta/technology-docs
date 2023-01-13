- Feature Name: `glides-trainsheets-servicechanges`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)
- Asana task: [asana link](https://app.asana.com/)
- Status: Proposed

# Summary
[summary]: #summary

| GTFS-realtime | Realtime Edits |
|---------------|----------------|
|Glides will provide trainsheet-based service data to RTR and other services that need it by providing a [GTFS-ServiceChanges](http://bit.ly/gtfs-service-changes-v3_1) JSON feed via Amazon S3.|Glides will provide trainsheet-based service data to RTR and other services that need it by sending [CloudEvents](https://cloudevents.io/)-formatted data to applications that wish to consume that data.|

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

it uses a GTFS-realtime TripUpdate event (with a nonstandard `consist` field) to convey that information in the feed:
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
  "type": "com.mbta.ctd.realtime-edits.trip-assignment.v1",
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

Glides uses a GTFS-realtime TripUpdate event to mark the trip as canceled:
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
  "type": "com.mbta.ctd.realtime-edits.dropped-trip.v1",
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

Glides uses a GTFS-realtime TripUpdate event (with a nonstandard `revenue` field) to mark the trip as a duplicate of an existing trip running the same path:
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
      "start_time": "15:57:09",
      "revenue": true
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
  "type": "com.mbta.ctd.realtime-edits.added-trip.v1",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "template-trip-id": "50974586",
    "new-trip-id": "757460c22ece",
    "service-date": "2022-04-12",
    "start-time": "15:57:09",
    "revenue": true
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

Glides uses a GTFS-realtime TripUpdate event to mark the applicable stops as skipped:
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
  "type": "com.mbta.ctd.realtime-edits.adjusted-trip.v1",
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

Glides uses a [GTFS-ServiceChanges v3.1](https://bit.ly/gtfs-service-changes-v3_1) NewTrips event to provide the new sequence of stops and a GTFS-ServiceChanges v3.1 TripUpdate event to mark the original trip as replaced by the new trip:
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

(Times should in principle not be there, but they're required in GTFS-NewTrips as of GTFS-ServiceChanges v3.1.)

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

Glides sends a restructured trip event to consumers:
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.realtime-edits.restructured-trip.v1",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50973989",
    "stops": [
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

## Adjusted Trips - Revenue Status Change

If a trip's revenue/non-revenue status needs to be changed,
<table>
<tr>
<th>GTFS-realtime</th>
<th>Realtime Edits</th>
</tr>
<tr>
<td>

Glides uses a GTFS-realtime TripUpdate event (with a nonstandard `revenue` field) to mark the trip with its new revenue state:
```json
{
  "id": "b9a1e9112231",
  "timestamp": 1649103580,
  "trip_update": {
    "trip": {
      "trip_id": "50974586"
    },
    "trip_properties": {
      "revenue": false
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
  "type": "com.mbta.ctd.realtime-edits.adjusted-trip.v1",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2022-04-12T13:16:10-05:00",
  "data": {
    "trip-id": "50974586",
    "revenue": false
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
  "type": "com.mbta.ctd.realtime-edits.start-time.v1",
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
  "type": "com.mbta.ctd.realtime-edits.hold-time.v1",
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

## Alternative: protobuf instead of JSON

GTFS-realtime is [canonically defined](https://github.com/google/transit/tree/master/gtfs-realtime/spec/en#data-format) using Google's protobuf binary encoding format, and so we could use that encoding instead of JSON for Glides trainsheet data.
However, the only advantage to this encoding over JSON is bandwidth usage, and we do not need to optimize for bandwidth usage between servers.
As such, JSON as a human-readable and easily extensible encoding makes more sense here than protobuf does.

# Prior art
[prior-art]: #prior-art

The CloudEvents format was selected in [RFC 4](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0004-socket-proxy-ocs-cloudevents.md) for passing OCS data from Trike to RTR.
Amazon Kinesis was selected in [RFC 5](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0005-kinesis-proxy-json.md) for passing OCS data from Trike to RTR.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?
-->

- should undrop trip be a separate event in realtime edits or should we change drop trip to cover both use cases?

## GTFS-realtime + GTFS-ServiceChanges vs Realtime Edits

### GTFS-realtime + GTFS-ServiceChanges

Advantages:
- Can start driving broader adoption of GTFS-ServiceChanges and maybe get GTFS-NewTrips added to the GTFS-realtime specification itself

Non-advantages:
- Data is already in the right format to be incorporated into public-facing predictions (not an advantage because data will need to be post-processed and incorporated into calculated predictions anyway, so it can't be used verbatim, making its initial format less relevant)

Disadvantages:
- Less responsive to newly discovered requirements that aren't met by the current standard/proposal (we can add our own fields if we have to, but that defeats the purpose of sticking with the standard/proposal)
- Not designed around partial records: we only use certain fields, but the sets of fields we use in various circumstances become complicated to manage
- No mechanism for specifying revenue status of trips, since GTFS is designed for public-facing revenue data only

### Realtime Edits

Advantages:
- Explicit event types that match our needs
- [Information is preserved more or less as entered and can be passed to e.g. Lamp without having to reconstruct the user's intent from GTFS-realtime data](https://github.com/mbta/technology-docs/pull/12#issuecomment-1093090735)

Disadvantages:
- Can't be used as is for vendor communication
- Does not advance GTFS-ServiceChanges spec proposal

## Pull/Push

In terms of how trainsheets data from Glides gets to RTR and other services, there are three main options: Glides publishes a feed to Amazon S3 that RTR pulls on an interval, Glides pushes events to RTR directly via an HTTP POST request, or Glides pushes events to RTR via an event broker like Amazon Kinesis or Amazon SQS.

### Pull

Glides stores a feed containing all currently-relevant information in Amazon S3 and RTR and other consumer services poll that feed on some schedule.

Advantages:
- Straightforward to manually inspect current feed data
- Straightforward to connect RTR development instances to Glides production data
- Matches GTFS-realtime architecture (only relevant if using GTFS-realtime as interchange format)

Disadvantages:
- RTR etc have to tune polling interval to get data at a reasonable pace without wasting resources

### Direct Push

When a Glides user makes a relevant change to trainsheets, Glides sends an HTTP POST request to RTR and other consumer services.

Advantages:
- New data is immediately presented to consumer services

Disadvantages:
- Glides need a hand-maintained list of URLs to POST to, so new consumers require maintenace effort from Glides, and restructured consumers require deployment coordination
- Needs access controls which we'd have to implement ourselves

### Event Broker Push

When a Glides user makes a relevant change to trainsheets, Glides submits an event to an Amazon Kinesis event stream or an Amazon SQS event queue.

Advantages:
- New data is promptly presented to consumer services

Disadvantages:
- Amazon Kinesis is designed for event streams with a substantially larger throughput than will be present here, so it might be overkill for this use case
- Amazon SQS is designed for messages that only need to be consumed once (only relevant if there may be more than one consumer service)

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
