- Feature Name: `glides-trainsheet-events`
- Start Date: 2023-01-20
- RFC PR: [mbta/technology-docs#18](https://github.com/mbta/technology-docs/pull/18)
- Status: Proposed

# Summary
Glides will provide high-level trainsheet information as JSON CloudEvents in a Kinesis stream.

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

## Scope
The stream of events proposed by this RFC serves two purposes:
- primarily, improve the rider-facing predictions produced by RTR, especially for departure times at a terminals, which currently do not have departure predictions at all, and
- secondarily, other internal operations needs such as Lean or LAMP

The goal of this RFC is to provide an interface which represents all data currently (2023H1) in Glides which is confirming or making changes to revenue service. While some additional flexibility is included, features outside the current scope of Glides are also outside the scope of these events and this RFC. Additionally, events which do not refer to light-rail service are also outside the scope of this RFC.

## CloudEvents
All events will be in the [CloudEvents v1.0.2](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) format (or later), using the JSON encoding. The event types will be under the `com.mbta.ctd.glides` event namespace, use `snake_case`, and will be in the past-tense (i.e. `trip_added` rather than `add_trip`).

## Kinesis
Events will be written as records to a Kinesis stream. Each Glides environment (`dev`, `dev-green`, and `prod`) will have a separate stream, named `ctd-glides-<environment>`. 

The partition key will be a hash of the station at which the inspector is working and the inspector's identity, which ensures that multiple events from a single inspector are ordered correctly if the records are distributed across multiple shards.

As the stream will include potential PII (operator badge information) it MUST be configured as encrypted-at-rest. 

Multiple events can be included in a single Kinesis record, by wrapping them in a JSON array. This is an optimization for improving write speeds: further steps in the Kinesis pipeline may break up or rearrange these arrays. If multiple events are part of the same user action, the [Batched](#Batched) event SHOULD be used to combine them.

## Value types

### Author
Each event which is caused by a person will include an `author` key, indicating who performed the given intervention.

- `emailAddress` (string): the e-mail of the logged-in user.
- `badgeNumber` (string, optional): the badge number of the logged-in user.
- `location` (Location, optional): the location the logged-in user is managing.
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

### Delta T
A value that can either be an instance of T, `"unmodified"`, or `"cleared"`. For example, Delta Car can be
```json
{ "label": "3800"}
# or
"unmodified"
# or 
"cleared"
```

### Dropped Reason
- `reason` (string, required): free-text description about why a trip was dropped.
```json
{"reason": "staffing"}
```
It is represented as an objecto provide future extensibility if needed.

### Location
One of:
- `{"gtfsId": "<GTFS stop ID>"}`, where the GTFS `location_type` is 1 (station).
- `{"glidesId": "<other>"}`: Other unique identifier for non-revenue locations, internal to Glides.

### Operator
Operators will be described by their badge number:
```json
{
  "badgeNumber": "123"
}
```
It is represented as an object to provide future extensibility if needed.

### Time
Time in the `HH:MM:SS` format. The time is measured from "noon minus 12h" of the service day (effectively midnight except for days on which daylight savings time changes occur). For times occurring after midnight, enter the time as a value greater than 24:00:00 in `HH:MM:SS` local time for the day on which the trip schedule begins.  Effectively, a [GTFS Time](https://github.com/google/transit/blob/master/gtfs/spec/en/reference.md#field-types) but without support for `H:MM:SS` times.
> _Example: `04:30:00` for 4:30AM or `25:35:00` for 1:35AM on the next day._

### Trip Key
For scheduled trips, Glides does not have the same trip ID used by RTR. For added trips, while an ID is provided, it can be the case that an added trip from the Glides perspective is matched to a GTFS trip, in which case RTR will use the GTFS trip ID in publishing the information.

In order to reduce event duplication, a Trip Key is used to identify both added and scheduled trips.

**Added Trip**
- `serviceDate` (RFC3999 date): the service date for the trip.
- `glidesId` (string): unique value representing this trip for the given `serviceDate`.
```json
{"serviceDate": "2023-01-20", "glidesId": "ADDED-123"}
```
**Scheduled Trip**
- `serviceDate` (RFC3999 date): the service date for the trip.
- `startLocation` (Location): where the trip is scheduled to start.
- `endLocation` (Location): where the trip is scheduled to end.
- `startTime` (Time): when the trip is scheduled to depart `startLocation`.
- `endTime` (Time): when the trip is scheduled to arrive at `endLocation`.

```json
{
  "serviceDate": "2023-01-20",
  "startLocation": {"gtfsId": "place-mdftf"},
  "endLocation": {"gtfsId": "place-heath"},
  "startTime": "04:47:00",
  "endTime": "05:34:00"
}
```
*Notes*:
- The added/scheduled distinction here is internal to Glides, and may not align with what is in other data sources. For example, an Added Trip in Glides may still correspond to a scheduled GTFS trip, and a Scheduled Trip may not appear in GTFS (due to track closures, for example).

## Events

### Editors Changed
Operator sign in and out of editing specific trainsheets. This event publishes changes to who is editing. Since one action by an inspector might result in multiple changes, for example if they sign out of one location and in to another, the changes are published in a list.

In practice, we limit each location to have only one editor, and limit each editor to only one location, but that rule may or may not change and this data in this event doesn't guarantee that.

Note that the author may or may not match the editors in `changes`, since inspectors can change whether other people are editing. For example, when an inspector takes over editing at a location, they may sign out the previous editor. Then they would issue a `stop` change for the previous editor and a `start` change for themself.

Event type: `com.mbta.ctd.glides.editors-changed.v1`
Fields in the event:
- `author` (Author): the inspector who made the changes
- `changes` (array of EditorChange): a list of start and stop editing events

#### EditorChange
Fields in the object:
- `type` (string `"start"`|`"stop"`): Whether the editor started or stopped editing.
- `location` (Location): the location the editor started or stopped managing.
- `emailAddress` (string): the e-mail of the editor.
- `badgeNumber` (string, optional): the badge number of the editor.


### Operator signed in
At the start of their shift, operators need to confirm that they are fit-for-duty and do not have any electronic devices. They currently do this by physically signing a paper trainsheet: in the future, they will do this digitally.

Event type: `com.mbta.ctd.glides.operator_signed_in.v1`
Fields in the event:
- `author` (Author): the inspector who signed in the operator
- `operator` (Operator): the operator who signed in
- `signedInAt` (RFC3999 timestamp): the time at which they signed in (separate from the `time` of the event)
- `confirmation` (object): how the operator confirmed that they signed in. Some possibilities are their badge number (`{"type": "<badge number>"`}) or an RFID tag number (`{"rfid": "<RFID tag">}`). The specific formats may change: consumers SHOULD NOT depend on any specific format.

### Trips Updated
Glides has new information about trips. Consumers who want to know what trains are running should pay attention to this event, and don't need to pay attention to any other events.

Event type: `com.mbta.ctd.glides.trips_updated.v1`
Fields in the event:
- `author` (Author): the inspector who inputed the new information.
- `inputType` (string): the action the inspector performed in Glides that caused the update. There are many ways to update trips in Glides, and all updated information normalized into the same format in `tripUpdates`. This field indicates what the inspector was doing in Glides, for consumers who care not just about service, but also care about how inspectors are using Glides. Example values are `"add-trip"` or `"manage-headways"`, but no guarantees are given about what strings will be used.
- `tripUpdates` (array of (TripUpdated|TripAdded)): The list of one or more trips that have new information.

#### TripUpdated
This indicates that a trip was updated in some fashion:  consist, operators, departure time. The trip is either a scheduled trip, or an added trip that has already appeared in the events stream. Fields which are not present are not considered to be updated.

Fields in the object:
- `type` (string): `"updated"|"added"`. Determines whether this is a `TripUpdated` object or a `TripAdded` object. If it's `"added"`, then it's a `TripAdded` object, see below. Subsequent updates to previously-added trips have `type` `"updated"`.
- `tripKey` (Trip Key): which trip is being updated.
- `comment` (string, optional): free text information about the trip.
- `startLocation` (Location, optional): the new destination of the train.
- `endLocation` (Location, optional): the new destination of the train.
- `startTime` (Time, optional): if present, the new time that the train is expected to depart `startLocation` (or the existing `startLocation` of the trip).
- `endTime` (Time, optional): if present, the new time that the train is expected to arrive at `endLocation` (or the existing `endLocation` of the trip).
- `consist` (array of Delta Car, optional): the cars assigned to perform this trip. If a car is `"cleared"`, then no car is currently assigned to that position in the consist. If a car is `"unmodified"` then the car is the value from the most-recent [Trip updated](#Trip-updated) event. The front car is listed first.
- `scheduledOperators`: (array of Operator, optional): if present, the operators who were scheduled to operate this trip.
- `operators` (array of Delta Operator, optional): the list of operators assigned to perform this trip. If an operator is `"cleared"`, then no operator is assigned to the car in that position in the consist. If the operator is `"unmodified"` then the operator is the value from the schedule or the most-recent [Trip updated](#Trip-updated) event. The operator of the front car is listed first.
- `dropped`: (Dropped Reason or `false`, optional): if `false`, the trip is not dropped (and restored if previously dropped). If a [Dropped Reason](#dropped-reason), the reason the trip was dropped.

*Notes*
- setting a new location or time does not modify the [Trip Key](#trip-key) for a scheduled trip.
- `consist` and `operators` MUST be the length of the train: length 1 for a 1-car consist, length 2 ted

#### TripAdded
There is a new trip, that does not appear in the Glides' schedule data. Any added trips will appear exactly once a TripAdded object, and any future updates will appear as TripUpdated objects.
"Added" is relative to the schedule that Glides uses, and an added trip may correspond to a trip that appears in a different schedule.

The event SHOULD only include values which were explicitly included by the `author`: inferred values SHOULD NOT be included.

TripAdded objects are the same as TripUpdated, with the following changes, so that the new trip can be placed in the appropriate order within existing trips:

New fields:
- `nextTripKey` (Trip Key, optional): links the newly created trip to the trip after it. If the Trip Key refers to an added trip, it may not have been seen in the event stream at the time of this event. Required if `startLocation` is specified and `startTime` is NOT specified.
- `previousTripKey` (Trip Key, optional): links the newly created trip to the trip immediately before it. If the Trip Key refers to an added trip, it may not have been seen in the event stream at the time of this event. Required if `endLocation` is specified and `endTime` is NOT specified.

New restrictions on existing fields:
- `type` (string): MUST be `"added"`.
- `tripKey` (TripKey): will always be the added trip form, and never the scheduled trip form.
- `startLocation` (Location, conditionally required): where the trip will be starting. Required if `startTime` is specified.
- `endLocation` (Location, conditionally required): where the trip will be ending. Required if `endTime` is specified.
- At least one of `startLocation` and `endLocation` is required.
- If `startLocation` is present, then at least one of `startTime` or `previousTripKey` is required.
- If `endLocation` is present, then at least one of `endTime` or `nextTripKey` is required.


# Reference-level explanation
## Effects on RTR
These inputs are similar to what RTR is already receiving from OCS.
- Trip added: `TSCH NEW`
- Trip updated: `TSCH CON` (set consist), `TSCH DEL` (drop or restore a trip), `TSCH OFF` (change departure time)

One key difference is that the trainsheets do not record the destination or route for trips, so added trips currently only include one end of the trip. RTR SHOULD assume a default pattern for these trips.

## Short-term storage
Events are only maintained in Kinesis for 24 hours. Some events (such as a [Trip updated][#trip-updated] which modifies the operators) can affect trips more than 24 hours in the future. Clients SHOULD maintain records internally of these future changes. Clients MUST not expect that reading the Kinesis stream from the trim horizon will return all events affecting the current service day.

## Long-term storage / querying
For future use, events will be archived to S3, using Kinesis Firehose. They can either be sent to LAMP's existing folder, or a new folder. As the stream contains potential PII (operator badge information) any long-term storage (S3, database) MUST be encrypted at rest. Only Glides environments which map to a Data Platform / LAMP environment will have their events recorded.

## Schema
All events MUST have their schema documented in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository. Schemas SHOULD use [JSON Schema](https://json-schema.org/specification.html) 2020-12 or later.

## Compatibility
Changes to the events can happen in one of two ways: backwards-compatible or backwards-incompatible. Updating the RFC is not necessary for either type of event change, but the schema repository MUST be kept up to date when the events are changed.

### Backwards-compatible changes
Backwards compatible changes to an existing event `type` MUST meet two requirements:
- old events MUST still be meet the schema.
- the semantics of the event MUST not change.

An example of a backwards-compatible change to an existing event would be adding an optional `reason` field to [Trip added][#trip-added].

Adding a new event `type` (which is not replacing an existing event `type`) is backwards-compatible.

Backwards-compatible changes to an event `type` are documented by bumping the minor version in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository.

### Backwards-incompatible changes
Any other changes are backwards-incompatible. Producers can add a new event `type` with a `v2` (or `v3`, &c) suffix to make backwards-incompatible changes. Producers MUST not make backwards-incompatible changes to an existing event `type`.

Producers SHOULD continue to produce versions of the event while there are still consumers who can only interpret that version. Consumers SHOULD ignore earlier versions of the same event if they can parse a later version (i.e. ignoring `v1` events if they can parse `v2`).

### Consumer requirements

Consumers MUST ignore event `type`s that they do not understand.
Consumers MUST ignore fields in events which they do not understand, trusting that any additional fields are not required to process the event.

## Examples
### Managing headways
- Inspector Alice (badge number: 123) is working at Boston College
- There are four scheduled trips going to Government Center: 9:55am, 10:00am, 10:05am, and 10:10am
- Due to staffing issues, the 10:05am trip is dropped
- Using the "Manage Headways" feature, Alice changes the expected departure times of the remaining trips to 9:56am, 10:02am, and 10:08am.
```json
{
  "type": "com.mbta.ctd.glides.trip_dropped.v1",
  "specversion": "1.0",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-20T09:50:00-05:00",
  "data": {
    "author": {
      "emailAddress": "ainspector@example.com",
      "badgeNumber": "123",
      "location": {"gtfsId": "place-lake"}
    },
    "tripKey": {
      "serviceDate": "2022-01-20",
      "startLocation": {"gtfsId": "place-lake"},
      "endLocation": {"gtfsId": "place-gover"},
      "startTime": "10:05:00",
      "endTime": "10:52:00"
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
      "emailAddress": "ainspector@example.com",
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
            "startTime": "09:55:00",
            "endTime": "10:42:00"
          },
          "location": {"gtfsId": "place-lake"},
          "startTime": "9:56:00"
        }
      },
      {
        "type": "com.mbta.ctd.glides.trip_updated.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "startLocation": {"gtfsId": "place-lake"},
            "endLocation": {"gtfsId": "place-gover"},
            "startTime": "10:00:00",
            "endTime": "10:47:00"
          },
          "location": {"gtfsId": "place-lake"},
          "startTime": "10:02:00"
        }
      },
      {
        "type": "com.mbta.ctd.glides.trip_updated.v1",
        "data": {
          "tripKey": {
            "serviceDate": "2022-01-20",
            "startLocation": {"gtfsId": "place-lake"},
            "endLocation": {"gtfsId": "place-gover"},
            "startTime": "10:10:00",
            "endTime": "10:57:00"
          },
          "location": {"gtfsId": "place-lake"},
          "startTime": "10:08:00"
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
      "emailAddress": "ainspector@example.com",
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
      "startTime": "10:00:00",
      "endTime": "10:47:00"
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
      "startTime": "09:55:00",
      "endTime": "10:42:00"

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
          "startTime": "10:00:00",
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
      "emailAddress": "ainspector@example.com",
      "badgeNumber": "123",
      "location": {"gtfsId": "place-matt"}
    },
    "tripKey": {
      "serviceDate": "2023-01-22",
      "startLocation": {"gtfsId": "place-matt"},
      "endLocation": {"gtfsId": "place-ashmt"},
      "startTime": "25:30:00",
      "endTime": "25:38:00"
    },
    "location": {"gtfsId": "place-matt"},
    "startTime": "25:45:00"
  }
}
```

Note that the date of the the event is 2023-01-23, but the service date is 2023-01-22. The hour in the time itself is greater than 24 hours, to reflect that the time is the next calendar day.

# Drawbacks
- This requires Glides to include new functionality to write their events into a Kinesis stream. However, they already use `ex_aws` so this additional functionality should not be hard to add.
 
# Rationale and alternatives
- RTR already consumes a Kinesis stream of CloudEvents from OCS, so it's a limited additional lift to consume a new Kinesis stream.
- Kinesis only adds a limited (<1s) latency between when the event is generated and when it is available to consumers.
- Kinesis Firehose allows writing the stream of records to S3 for LAMP to consume, without needing a separate integration.

## Alternative: allow setting times at non-terminal locations
An earlier version of this version supported setting arrival/departure times for non-terminal locations as a part of the [Trip updated](#trip-updated) event. This is not supported by Glides, and there are no plans to support it: see [Future possibilities](#future-possibilities).

## Alternative: allow `location_type` 0 locations
While more flexible, it requires consumers of the events to support two types of GTFS stops. Glides has no plans to use the additional flexibility.

## Alternative: events for importing the schedule
Instead of the `scheduledOperators` field in [Trip updated](#trip-updated), another possibility would be including events which allow consumers to determine the scheduled operator. However, these events are complicated to implement, and require consumers to maintain state for many months (as the schedule events would not be generated frequently).

## Alternative: times as RFC3999 string
This would be easy to parse with standard tools, but doesn't match existing OCS behavior which only provides a time.

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

## Alternative: separate Dropped and Restored Trip events
Advantage:
- it disallows for confusing [Trip updated](#Trip-updated) events. How should clients interpret a [Trip updated](#Trip-updated) event which included `dropped`, along with other changes to the trip?
Disadvantages:
- requires two additional events

## Alternative: split update types into different events
Examples:
- set departure time
- set/unset operator
- set/unset consist

The current trainsheet interface does not distinguish between these cases: all changes are saved at the same time. Functionality to separate them would need to be added to Glides, or multiple events would need to be written in a batch, increasing duplication.

## Alternative: other levels of abstraction
The current level of abstraction for events represents a medium level of abstraction for changes to service. A higher level might be to have events for the "Manage Headways" feature, for example. However, there will always be a need for more granular events, as inspectors make changes to individual trips. Even more granular events (such as inspector login, or particular buttons being clicked) are not useful to consumers in interpreting the expected changes to service.

# Prior art
[Trike](https://github.com/mbta/trike) uses a similar approach to provide events to RTR for influencing predictions and to OCS Saver for future processing by LAMP / OPMI. These events come from the heavy-rail dispatching system, which uses another implementation of trainsheets. This approach is described in [RFC4](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0004-socket-proxy-ocs-cloudevents.md) and [RFC5](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0005-kinesis-proxy-json.md).

# Unresolved questions
- While CloudEvent formats are included here, the final specific format should be settled on as a conversation between the Glides, Transit Data, and LAMP teams.

# Future possibilities
- As Glides becomes more widely adopted, further information entered by Chief Inspectors and other officials can be included as events, such as short turns, holds, express trips, non-revenue trips, and yard management. However, this is not currently scheduled for implementation.
