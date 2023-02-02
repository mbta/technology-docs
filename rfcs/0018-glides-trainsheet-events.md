- Feature Name: `glides-trainsheet-events`
- Start Date: 2023-01-20
- RFC PR: [mbta/technology-docs#18](https://github.com/mbta/technology-docs/pull/18)
- Status: Proposed

# Summary
Glides will provide high-level trainsheet information as CloudEvents in a Kinesis stream

# Motivation
As part of digitizing the currently paper-based trainsheets that define operational conditions on the Green Line, Glides will be gathering information about consist assignments, dropped/added trips, and departure time adjustments from its users: pull-out inspectors at selected terminal stations. This information will be useful to other consumers: primarily RTR for use in predictions, and secondarily for other operations needs.

# Guide-level explanation
The Glides trainsheet functionality hews closely to what the paper-based trainsheets provide, along with some shortcuts for common actions.

- sign an operator in, marking them fit-for-duty
- add a trip (1-way from the current location, 1-way arriving at the current location, or shortcut: round-trip from the current location)
- drop a trip (with a reason)
- add/change/remove train cars to a trip
- add/change/remove the operators assigned to a car/trip
- set a new departure time for a trip
- shortcut: modifying multiple trips together to manage headways
- shortcut: removing an operator from multiple trips (due to illness, for example)

These actions are always taken by a logged-in user (generally an inspector).

There are additional processes which keep the schedule up-to-date:
- importing date/schedule mappings
- importing run assignments
- importing scheduled trips

The stream of events proposed by this RFC serves two purposes:
- primarily, improve the rider-facing predictions produced by RTR, especially for departure times at a terminals, which currently do not have departure predictions at all, and
- secondarily, other internal operations needs such as Lean or LAMP

The goal of this RFC is to provide an interface which represents all data currently (2023H1) in Glides. While some additional flexibility is included, features outside the current scope of Glides are also outside the scope of these events and this RFC. 

## CloudEvents
All events will be in the [CloudEvents v1.0.2](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) format (or later), using the JSON encoding. The event types will be under the `com.mbta.ctd.glides` event namespace, and will be in the past-tense (i.e. `trip_added` rather than `add_-_trip`).

## Kinesis
Events will be written as records to a Kinesis stream. Each Glides environment (`dev`, `dev-green`, and `prod`) will have a separate stream, named `ctd-glides-<environment>`. 

The partition key will be a hash of the station at which the inspector is working and the inspector's identity, which ensures that multiple events from a single inspector are ordered correctly if the records are distributed across multiple shards.

As the stream will include potential PII (operator badge information) it MUST be configured as encrypted-at-rest. 

Multiple events can be included in a single Kinesis record, by wrapping them in a JSON array. This is an optimization for improving write speeds: further steps in the Kinesis pipeline may break up or rearrange these arrays. If multiple events are part of the same user action, the [Batched](#Batched) event SHOULD be used to combine them.

## Value types

### Author
Each event which is caused by a person will include an `author` key, indicating who performed the given intervention.

- `emailAddress` (string): the e-mail of the logged-in user
- `badgeNumber` (string, optional): the badge number of the logged-in user
- `location` (Location, optional): the location the logged-in user is managing
```json
{
  "emailAddress": "name@example.com",
  "badgeNumber": "123456",
  "location": {"gtfsId": "place-lake"}
}

or

{
  "emailAddress": "pswartz@mbta.com",
  "location": {"gtfsId": "place-matt"}
}
```

### Car
Cars (individual parts of a train consist) will be described by their 4 digit number.
```json
{
  "label": "3800"
}
```
It is represented as an object to provide future extensibility if needed.

### Location
One of:
- `{"gtfsId": "<GTFS stop ID>"}`, where the GTFS `location_type` is 1 (station)
- `{"glidesId": "<other>"}`: Other unique identifier for non-revenue locations, internal to Glides

### Operator
Operators will be described by their badge number:
```json
{
  "badgeNumber": "123"
}
```
It is represented as an object to provide future extensibility if needed.

### Delta T
A value that can either be an instance of T, `"unmodified"`, or `"cleared"`. For example, Delta Car can be
```json
{ "label": "3800"}
# or
"unmodified"
# or 
"cleared"
```

### Time
A [GTFS Time](https://github.com/google/transit/blob/master/gtfs/spec/en/reference.md#field-types).
> Time in the HH:MM:SS format (H:MM:SS is also accepted). The time is measured from "noon minus 12h" of the service day (effectively midnight except for days on which daylight savings time changes occur). For times occurring after midnight, enter the time as a value greater than 24:00:00 in HH:MM:SS local time for the day on which the trip schedule begins.  
> _Example: `14:30:00` for 2:30PM or `25:35:00` for 1:35AM on the next day._

### Trip Key
For scheduled trips, Glides does not have the same trip ID used by RTR. For added trips, while an ID is provided, it can be the case that an added trip from the Glides perspective is matched to a GTFS trip, in which case RTR will use the GTFS trip ID in publishing the information.

In order to reduce event duplication, a Trip Key is used to identify both added and scheduled trips.

**Added Trip**
- `serviceDate` (RFC3999 date): the service date for the trip
- `glidesId` (string): unique value representing this trip for the given `serviceDate`
```json
{"serviceDate": "2023-01-20", "glidesId": "ADDED-123"}
```
**Scheduled Trip**
- `serviceDate` (RFC3999 date): the service date for the trip
- `startLocation` (Location): where the trip is scheduled to start
- `endLocation` (Location): where the trip is scheduled to end
- `departureTime` (Time): when the trip is scheduled to depart `startLocation`
- `arrivalTime` (Time): when the trip is scheduled to arrive at `endLocation`

```json
{
  "serviceDate": "2023-01-20",
  "startLocation": {"gtfsId": "place-mdftf"},
  "endLocation": {"gtfsId": "place-heath"},
  "departureTime": "4:47:00",
  "arrivalTime": "5:34:00"
}
```
*Notes*:
- The added/scheduled distinction here is internal to Glides, and may not align with what is in other data sources. For example, an Added Trip in Glides may still correspond to a scheduled GTFS trip, and a Scheduled Trip may not appear in GTFS (due to track closures, for example)

## Events
### Batched
If multiple events are generated as a part of the same author action, then they can be combined into a single Batched event. This ensures that all of the events are kept together throughout the data pipeline.

Event type: `com.mbta.ctd.glides.batched.v1`
Fields in the event:
- `author` (Author, optional): the human who performed the events, if it is the same for all the events in the `events` array. 
- `events` (array of `{"type": string, "data": JSON}`): each element in the array is a CloudEvent including only the `type` and `data` field, and the `data` fields of the individual events SHOULD NOT re-include the `author` field. See [Examples](#examples) for an example.

### Imported date schedules
These values map a service date to a schedule identifier for the given line. This event may happen at any time, and subsequent events will update the `scheduleId` used on the `serviceDate` for the `line` (but not the `scheduleId` used on the `serviceDate` for other lines).

Event type: `com.mbta.ctd.glides.imported_date_schedules.v1`
Fields in the event:
- `line` (string): the line for which a new date/schedule mapping is being imported
- `ratingName` (string): the rating to which the new dates belong
- `schedules`: Array of:
  - `serviceDate` (RFC3999 date): date on which the given schedule will be used
  - `scheduleId` (string): unique identifier for the schedule of trips running on this date 

### Imported run assignments
Runs map an operator to their work for a given `serviceDate`. Subsequent events update the `operator` assigned to the `runId` on the given `serviceDate`.

Event type: `com.mbta.ctd.glides.imported_run_assignments.v1`
Fields in the event:
- `runs`: Array of:
  - `serviceDate` (RFC3999 date)
  - `runId` (string): identifier for the run. MUST be unique for the given `serviceDate`.
  - `operator` (Operator)

### Imported scheduled trips
This event maps a given `scheduleId` to the trips running for that day. Subsequent events replace all the trips for the given `scheduleId`.

Event type: `com.mbta.ctd.glides.imported_scheduled_trips.v1`
Fields in the event:
- `scheduleId` (string)
- `trips`: Array of:
  - `route` (string)
  - `startLocation` (Location)
  - `departureTime` (Time)
  - `endLocation` (Location)
  - `arrivalTime` (Time)
  - `runs` (Array of `runId`): the runs operating this trip, with the lead car in the consist first.

*Notes*
- the combination of `startLocation`, `endLocation`, `departureTime`, and `arrivalTime` MUST be unique for the `scheduleId`.

### Operator signed in
At the start of their shift, operators need to confirm that they are fit-for-duty and do not have any electronic devices. They currently do this by physically signing a paper trainsheet: in the future, they will do this digitally.

Event type: `com.mbta.ctd.glides.operator_signed_in.v1`
Fields in the event:
- `author` (Author): the inspector who signed in the operator
- `operator` (Operator): the operator who signed in
- `signedInAt` (RFC3999 timestamp): the time at which they signed in (separate from the `time` of the event)
- `confirmation` (object): how the operator confirmed that they signed in. Some possibilities are their badge number (`{"type": "<badge number>"`}) or an RFID tag number (`{"rfid": "<RFID tag">}`)

### Trip added
This creates a new trip in the timesheet. It may or may not be a new trip relative to GTFS (see the splitting a train reference example).

Event type: `com.mbta.ctd.glides.trip_added.v1`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): provides a unique identifier for the newly created trip
- `route` (string, optional): if specified, the route tag for the trip. This is different from a GTFS route ID.
- `startLocation` (Location, conditionally required): where the trip will be starting. Required if `departureTime` is specified.
- `endLocation` (Location, conditionally required): where the trip will be ending. Required if `arrivalTime` is specified.
- `departureTime` (Time, conditionally required): when the added trip is expected to depart `startLocation`.
- `arrivalTime` (Time, conditionally required): when the added trip is expected to arrive at `endLocation`.
- `previousTripKey` (Trip Key, optional): links the newly created trip to the trip immediately before it. If the Trip Key refers to an added trip, it may not have been seen in the event stream at the time of this event. Required if `endLocation` is specified and `arrivalTime` is NOT specified.
- `nextTripKey` (Trip Key, optional): links the newly created trip to the trip after it. If the Trip Key refers to an added trip, it may not have been seen in the event stream at the time of this event. Required if `startLocation` is specified and `departureTime` is NOT specified.
- `consist` (array of (Car|`null`), optional, default: empty list): the cars assigned to perform this trip. If a car is `null`, then no car is currently assigned to that position in the consist. 
- `operators` (array of (Operator|`null`), optional, default: empty list): the array of operators assigned to perform this trip. If an operator is null, then no operator is assigned to the car in that position in the consist.
- `comment` (string, optional): free text with additional information about the trip

*Notes*
- At least one of `startLocation` and `endLocation` is required.
- If `startLocation` is present, then at least one of `departureTime` or `previousTripKey` is required.
- If `endLocation` is present, then at least one of `arrivalTime` or `nextTripKey` is required.
- The event SHOULD only include values which were explicitly included by the `author`: inferred values SHOULD NOT be included.
- `consist` and `operators` MUST be the length of the train: length 1 for a 1-car consist, length 2 for a 2-car consist.

### Trip dropped
This removes a trip (either added or scheduled), indicating that it will not be running.

Event type: `com.mbta.ctd.glides.trip_dropped.v1`
Fields in the event:
- `author` (Author): the inspector dropping the trip.
- `tripKey` (Trip Key): which trip to drop. can be either an added or scheduled trip.
- `reason` (string): a text description of why the trip is being dropped.

### Trip restored
This restores a trip (either added or scheduled) that was previously dropped.

Event type: `com.mbta.ctd.glides.trip_restored.v1`
Fields in the event:
- `author` (Author): the inspector restoring the trip.
- `tripKey` (Trip Key): which trip to restore. can be either an added or scheduled trip.

### Trip updated
This indicates that a trip was updated in some fashion:  consist, operators, departure time. This includes updates which confirm an arrival or departure time. Fields which are not present are not considered to be updated.

Event type: `com.mbta.ctd.glides.trip_updated.v1`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): which trip is being updated
- `comment` (string, optional): free text information about the trip
- `endLocation` (Location, optional): the new destination of the train
- `location` (Location, optional): where the time/consist is being updated for. This can be different from the location in the `author` value or in the `tripKey`, if the author is updating information about the train relative to a different location. Required if either `arrivalTime` or `departureTime` are specified.
- `arrivalTime` (Time, optional): if present, the new time that the train is expected to arrive at the given location.
- `departureTime` (Time, optional): if present, the new time that the train is expected to depart the given location.
- `consist` (array of Delta Car, optional): the cars assigned to perform this trip. If a car is `"cleared"`, then no car is currently assigned to that position in the consist. If a car is `"unmodified"` then the car is the value from the most-recent [Trip updated](#Trip-updated) event. The front car is listed first.
- `operators` (array of Delta Operator, optional): the list of operators assigned to perform this trip. If an operator is `"cleared"`, then no operator is assigned to the car in that position in the consist. If the operator is `"unmodified"` then the operator is the value from the schedule or the most-recent [Trip updated](#Trip-updated) event. The operator of the front car is listed first.

*Notes*
- setting a new `endLocation` does not modify the [Trip Key](#trip-key) for a scheduled trip, nor does setting a `departureTime` or `arrivalTime` at one of the endpoints.
- `consist` and `operators` MUST be the length of the train: length 1 for a 1-car consist, length 2 for a 2-car consist.

# Reference-level explanation
## Effects on RTR
These inputs are similar to what RTR is already receiving from OCS.
- Trip added: `TSCH NEW`
- Trip dropped: `TSCH DEL` with delete status 1
- Trip restored: `TSCH DEL` with delete status 0
- Trip updated: `TSCH CON` (set consist), `TSCH OFF` (change departure time)

One key difference is that the trainsheets do not record the destination or route for trips, so added trips currently only include one end of the trip. RTR SHOULD assume a default pattern for these trips.

## Long-term storage / querying
For future use, events will be archived to S3, using Kinesis Firehose. They can either be sent to LAMP's folder, or a new folder. As the stream contains potential PII (operator badge information) any long-term storage (S3, database) MUST be encrypted at rest.

### Schema
All events MUST have their schema documented in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository.

### Compatibility
Changes to the events can happen in one of two ways:
- backwards-compatible changes (such as adding a new field with a default value) where old consumers can still interpret new events can be made while keeping the same event type. 
- incompatible-changes require creating a new event type, `<type>.v2` (creating a new version). Each version MUST be documented separately in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository.

Consumers MUST ignore event types that they do not understand.

Updating the RFC is not necessary for either type of event change, but the schema repository MUST be kept up to date if the events are changed.

## Examples
### Managing headways
- Inspector Alice (badge number: 123) is working at Boston College
- There are four scheduled trips going to Government Center: 9:55am, 10:00am, 10:05am, and 10:10am
- Due to staffing issues, the 10:05am trip is dropped
- Using the "Manage" headways, Alice changes the expected departure times of the remaining trips to 9:56am, 10:02am, and 10:08am.
```json
{
  "type": "com.mbta.ctd.glides.trip_dropped.v1",
  "specversion": "1.0",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-20T09:50:00-05:00",
  "data": {
    "author": {
      "emailAddress": "ainspector",
      "badgeNumber": "123",
      "location": {"gtfsId": "place-lake"}
    },
    "tripKey": {
      "serviceDate": "2022-01-20",
      "startLocation": {"gtfsId": "place-lake"},
      "endLocation": {"gtfsId": "place-gover"},
      "departureTime": "10:05:00",
      "arrivalTime": "10:52:00"
    },
    "reason": "staffing"
  }
}
# specversion, source, id elided
{
  "type": "com.mbta.ctd.glides.batched.v1",
  "time": "2023-01-20T09:50:10-05:00",
  "data": {
    "author": {
      "emailAddress": "ainspector",
      "badgeNumber": "123",
      "location": {"gtfsId": "place-lake"}
    },
    "events": [
      {
        "type": "com.mbta.ctd.glides.trip_updated.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "startLocation": {"gtfsId": "place-lake"},
            "endLocation": {"gtfsId": "place-gover"},
            "departureTime": "09:55:00",
            "arrivalTime": "10:42:00"
          },
          "departureTime": "9:56:00"
        }
      },
      {
        "type": "com.mbta.ctd.glides.trip_updated.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "startLocation": {"gtfsId": "place-lake"},
            "endLocation": {"gtfsId": "place-gover"},
            "departureTime": "10:00:00",
            "arrivalTime": "10:47:00"
          },
          "departureTime": "10:02:00"
        }
      },
      {
        "type": "com.mbta.ctd.glides.trip_updated.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "startLocation": {"gtfsId": "place-lake"},
            "endLocation": {"gtfsId": "place-gover"},
            "departureTime": "10:10:00",
            "arrivalTime": "10:57:00"
          },
          "departureTime": "10:08:00"
        }
      }
    ]
  }
}
```

### Splitting a 2-car train into two 1-car trips
- Inspector Alice (badge number: 123) is working at Boston College
- There are two scheduled trips: 9:55am and 10:00am
- Operator A (badge number: 456) and Operator B (badge number: 567) are scheduled to perform the 9:55am
- Only 1 trainset (3800 and 3850) is available
- In order to maintain headways, Inspector Alice will split the trainset
- Operator A will take 3800 and run the 9:55am
- Operator B will take 3850 and run the 10:00am

```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.glides.operator-signed-in.v1",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-20T09:45:10-05:00",
  "data": {
    "author": {
      "emailAddress": "ainspector",
      "badgeNumber": "123",
      "location": {"gtfsId": place-lake"}
    },
    "operator": {
      "badgeNumber": "456"
    },
    "signedInAt": "2023-01-20T09:45:00-05:00"
  }
}
# further specversion, source, time, and ID fields elided from the top level
# author elided from underneath "data"
{
  "type": "com.mbta.ctd.glides.operator-signed-in.v1",
  "data": {
    "operator": {
      "badgeNumber": "567"
    },
    "signedInAt": "2023-01-20T09:45:00-05:00"
  }
}
{
  "type": "com.mbta.ctd.glides.trip_dropped.v1",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
      "startLocation": {"gtfsId": "place-lake"},
      "endLocation": {"gtfsId": "place-gover"},
      "departureTime": "10:00:00",
      "arrivalTime": "10:47:00"
    },
    "reason": "ran as single"
  }
}
{
  "type": "com.mbta.ctd.glides.trip_updated.v1",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
   	  "startLocation": {"gtfsId": "place-lake"},
      "endLocation": {"gtfsId": "place-gover"},
      "departureTime": "09:55:00",
      "arrivalTime": "10:42:00"

    },
    "operators": [
      {"badgeNumber": "456"}
    ],
    "consist": [
      {"label": "3800"}
    ],
    "comment": "single"
  }
}
{
  "type": "com.mbta.ctd.glides.batched.v1",
  "time": "2023-01-20T09:50:15-05:00",
  "data": {
    "events": [
      {
        "type": "com.mbta.ctd.glides.trip_added.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "id": "ADDED-1"
          },
          "startLocation": {"gtfsId": "place-lake"},
          "departureTime": "10:00:00",
          "nextTripKey": {
            "serviceDate": "2022-01-20",
            "id": "ADDED-2"
          },
          "consist": [
            {
              "label": "3850"
            }
          ],
          "operators": [
            {
              "badgeNumber": "567"
            }
          ],
          "comment": "single"
        }
      },
      {
        "type": "com.mbta.ctd.glides.trip_added.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "id": "ADDED-2"
          },
          "endLocation": {"gtfsId": "place-lake"},
          "previousTripKey": {
            "serviceDate": "2022-01-20",
            "id": "ADDED-1"
          },
          "consist": [
            {
              "label": "3850"
            }
          ],
          "operators": [
            {
              "badgeNumber": "567"
            }
          ],
          "comment": "single"
        }
      }
    ]
  }
}
```

In this instance, the inspector dropped the 10:00am trip, and created a new trip with the same departure time. Ideally, RTR will treat this the same as if the 10:00am were updated to be a 1-car trip, but inspectors do not always enter the data in that fashion.

### Train departing 15 minutes later than scheduled
- Inspector Alice (badge number: 123) is working at Mattapan
- The 1:30am trip is being held 15 minutes to wait for a final Red Line train
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.glides.trip_updated.v1",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-23T01:25:00-05:00",
  "data": {
    "author": {
      "emailAddress": "ainspector",
      "badgeNumber": "123",
      "location": {"gtfsId": "place-matt"}
    },
    "tripKey": {
      "serviceDate": "2023-01-22",
      "startLocation": {"gtfsId": "place-matt"},
      "endLocation": {"gtfsId": "place-ashmt"},
      "departureTime": "25:30:00",
      "arrivalTime": "25:38:00"
    },
    "departureTime": "25:45:00"
  }
}
```

Note that the date of the the event is 2023-01-23, but the service date is 2023-01-22. The hour in the time itself is greater than 24 hours, to reflect that the time is the next calendar day.

# Drawbacks
- This requires Glides to include new functionality to write their events into a Kinesis stream. However, they already use `ex_aws` so this additional functionality should not be hard to add.
 
# Rationale and alternatives
- RTR already consumes a Kinesis stream of CloudEvents from OCS, 
- so it's a limited additional lift to consume a new Kinesis stream
- Kinesis Firehose allows writing the stream of records to S3 for LAMP to consume, without needing a separate integration

## Alternative: departure time as RFC3999 string
This would be easy to parse with standard tools, but doesn't match existing OCS behavior which only provides a time.

## Alternative: `dropped` boolean in Trip updated
Advantage:
- eliminates the need for the [Trip dropped](#Trip-dropped) and [Trip restored](#Trip-restored) events
Disadvantages:
- it also allows for confusing [Trip updated](#Trip-updated) events. How should clients interpret a [Trip updated](#Trip-updated) event which included `dropped` True, along with other changes to the trip?
- Does not provide a location for the `reason` a trip was dropped

## Alternative: allow `location_type` 0 locations
While more flexible, it requires consumers of the events to support two types of GTFS stops. Glides has no plans to use the additional flexibility.

## Alternative: split update types into different events
Examples:
- set departure time
- set/unset operator
- set/unset consist

The current trainsheet interface does not distinguish between these cases: all changes are saved at the same time. Functionality to separate them would need to be added to Glides, or multiple events would need to be written in a batch, increasing duplication.

## Alternative: GTFS-RT ServiceChange feed
For much of CTD's history, we have used versions of GTFS-RT as our interchange format between applications. Current examples include:
- Operator assignments and block waivers (Busloc -> Skate)
- Arrival predictions at skipped stops (RTR -> Realtime Signs)

If this data can be directly presented to riders, this avoids a processing step.

However, this is also limiting:
- both sides must already understand GTFS
- can be hard to reproduce user intent
- expects full (or at least mostly full) records

While the ladder view in Glides understands GTFS-RT, the trainsheet information comes directly from HASTUS on a quarterly basis. It does not have updated trip IDs, so RTR would need to process the data anyways: it could not be provided directly to riders.

# Prior art
[Trike](https://github.com/mbta/trike) uses a similar approach to provide events to RTR for influencing predictions and to OCS Saver for future processing by LAMP / OPMI. This approach is described in [RFC4](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0004-socket-proxy-ocs-cloudevents.md) and [RFC5](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0005-kinesis-proxy-json.md).

# Unresolved questions
- While CloudEvent formats are included here, the final specific format should be settled on as a conversation between the Glides, Transit Data, and LAMP teams.
- How should scheduled operators be represented? The current set of `imported_*` events are meant to represent this, but have their own issues. Other possibilities include:
  - a `scheduledOperators` field in [Trip updated](#trip-updated)
  - a single event representing the schedule import (instead of the 3 events currently described)
  - a single event on some cadence, with future trips and their scheduled operators
- Some trainsheet entries (removing an operator from their trips, and the schedule) can happen more than 24 hours before the event's effect takes place. Do we need to persist any events longer than the 24 hour Kinesis default? Kinesis can store events up to a maximum of 7 days.

# Future possibilities
- As Glides becomes more widely adopted, further information entered by Chief Inspectors and other officials can be included as events, such as short turns, express trips, and non-revenue trips. However, this is not currently scheduled for implementation.
