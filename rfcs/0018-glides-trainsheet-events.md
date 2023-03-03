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

Multiple events can be included in a single Kinesis record, by wrapping them in a JSON array. This is an optimization for improving write speeds: further steps in the Kinesis pipeline may break up or rearrange these arrays.

## Value types

### Car
Fields:
- `label` (string | `"none"`, optional): car number corresponding to the GTFS-RT `label` field. Or `"none"` if the inspector has unassigned the car. If the field is absent, it is not modified.
- `operator` ([Operator](#Operator) | `"none"` | `"unset"`, optional): the operator assigned to this car. The string `"none"` means the inspector has unassigned the operator without reassigning another. The string `"unset"` means to discard the previous value as if the field had never been modified. If the field is absent, it is not modified from its previous value.

`"unset"` and `"none"` are semantically different, and used consistently with the `"unset"` values in [TripUpdated](#TripUpdated). If an operator is `"unset"`, then, as if no update had ever been made, clients SHOULD assume the scheduled operator will operate the car. If an operator is `"none"`, then we don't know who will operate the car.

An empty object `{}` is valid if nothing about the car has been modified.

The length of `cars` always reflects the known length of the train.

### DroppedReason
An object representing the reason that a trip was dropped. The presence of this object implies that the trip was dropped.

Fields:
- `reason` (string): free-text description about why a trip was dropped.

Currently the reason is entered by inspectors in a free-text field. It's published in an object so that if Glides collects more structured data in the future, then extra fields can be added. If more fields are added, then a generated text description would be filled into the `reason` field, and it would not be a breaking change.

`reason` MAY be the empty string if for some reason Glides does not have information about why a trip was dropped.

### EditorChange
Fields in the object:
- `type` (string `"start"`|`"stop"`): Whether the editor started or stopped editing.
- `location` ([Location](#Location)): the location the editor started or stopped managing.
- `editor` ([GlidesUser](#GlidesUser)): the affected editor.

### GlidesUser
Fields:
- `emailAddress` (string): the e-mail of the user.
- `badgeNumber` (string, optional): the badge number of the user. Not all Glides users have a badge number (for example, CTD contractors).

### Location
One of:
- `{"gtfsId": "<GTFS stop ID>"}`, where the GTFS `location_type` is 1 (station).
- `{"glidesId": "<other>"}`: Other unique identifier for non-revenue locations, internal to Glides.

### Metadata
Information about how the event was created by Glides. All important outputs from Glides will be elsewhere in the event data, but data here might be useful for consumers who care about how inspectors are using Glides.

Fields:
- `author` ([GlidesUser](#GlidesUser), optional): The logged-in user whose action triggered the event.
- `inputTimestamp` (RFC3999 timestamp, optional): The timestamp that the user entered the data into Glides, as reported by their device. This is different from the `time` field in the event, which is the server-reported time that the Glides server received the input. They could differ if sending the input was delayed due to network problems, or if the device's clock is wrong. Usually, you will want the event time instead, because Glides applies changes in the order they were received.
- `inputType` (string, optional): the action that the user did in Glides. This field exists because the event data on its own may not specify how the data was entered, for example all trainsheet updates are normalized into a generic format in `tripUpdates`. Example values for this field are `"add-trip"` or `"manage-headways"`, but no guarantees are given about what strings will be used or which actions the field will be populated for.
- `location` ([Location](#Location), optional): the location the logged-in user is managing.

### Operator
Operators will be described by their badge number:

Fields
- `badgeNumber` (string)

It is represented as an object to provide future extensibility if needed.

### Scheduled
Scheduled information about the trip. It was not updated in Glides, but is included in the event stream so that consumers can know the schedule information that Glides uses. If data here was never overriden by a corresponding field in [TripUpdated](#TripUpdated), then consumers can assume that the trip operated based on the scheduled information contained here.

Fields:
- `scheduledCars` (array of [ScheduledCar](#ScheduledCar)): Array of length 1 or 2. The length reflects the scheduled length of the train. In a 2 car train, the front car is listed first.

It does not include time and location fields, because for scheduled trips those fields are already in [TripKey](#TripKey)

### ScheduledCar
Fields:
- `run` (string, optional): The run number scheduled to the trip.
- `operator` ([Operator](#Operator), optional): The operator who is scheduled to operate the run that day, and therefore scheduled to do the trip.

If fields are missing from ScheduledCar, it is not known who is scheduled to operate that car. An empty object is valid if we know the car is scheduled to operate but don't know anything else about it.

### Signature
Represents the operator's signature in Glides confirming that they were fit for duty.

Fields
- `type` (string): How the the operator made their signature.

Current values for `type` are
- `"tap"` if they signed in by tapping their badge to an RFID scanner.
- `"type"` if they signed in by typing their badge number into Glides as a signature.

Represented as an object so we could include other types of signatures or other data about the signature in the future without a breaking change.

The tapped badge's RFID serial number is not included in the stream for security reasons (it could be used to impersonate an employee).

### Time
Time in the `HH:MM:SS` format. The time is measured from "noon minus 12h" of the service day (effectively midnight except for days on which daylight savings time changes occur). For times occurring after midnight, enter the time as a value greater than 24:00:00 in `HH:MM:SS` local time for the day on which the trip schedule begins. Effectively, a [GTFS Time](https://github.com/google/transit/blob/master/gtfs/spec/en/reference.md#field-types) but without support for `H:MM:SS` times.
> _Example: `04:30:00` for 4:30AM or `25:35:00` for 1:35AM on the next day._

### TripAdded
There is a new trip, that does not appear in the Glides' schedule data. "Added" is relative to the schedule that Glides uses, and an added trip may correspond to a trip that appears in a different schedule.

Any added trips SHOULD appear exactly once as a TripAdded object, and any future updates SHOULD appear as TripUpdated objects, (except in case of a duplicate event due to a network error, in which case the two events SHOULD have the same TripKey and data). Clients SHOULD ignore TripAdded events with a TripKey that has already been added.

The event SHOULD only include values which were explicitly included by the author: inferred values SHOULD NOT be included.

TripAdded objects are the same as TripUpdated, with the following changes, so that enough is specified about the trip to know or infer where and when the trip will happen:

New fields:
- `previousTripKey` ([TripKey](#TripKey), conditionally required): The added trip will happen immediately after the trip referred to by `previousTripKey`. Required if `startTime` and `endTime` are both not set. (If no time is specified, then the previous trip is necessary so the trip's time can be inferred.) This field MAY refer to a trip that has not been already seen in the event stream.

New restrictions on existing fields:
- `type` (string): MUST be `"added"`.
- `tripKey` ([TripKey](#TripKey)): will always be the added trip form, and never the scheduled trip form.
- `startLocation` ([Location](#Location), conditionally required): where the trip will start. Required if `startTime` is specified.
- `endLocation` ([Location](#Location), conditionally required): where the trip will end. Required if `endTime` is specified.
- At least one of `startLocation` and `endLocation` is required.
- At least one of `startTime`, `endTime`, and `previousTripKey` is required.
- `dropped` SHOULD NOT be set.
- All data SHOULD NOT be `"none"` or `"unset"`. (Instead, the fields would not be included).

Glides SHOULD include the `cars` field to indicate the length of the train, even if no information is known about each car.

### TripKey
For scheduled trips, Glides does not have the same trip ID used by RTR. For added trips, while an ID is provided, it can be the case that an added trip from the Glides perspective is matched to a GTFS trip, in which case RTR will use the GTFS trip ID in publishing the information.

In order to reduce event duplication, a TripKey is used to identify both added and scheduled trips.

**Added Trip**
- `serviceDate` (RFC3999 date): the service date for the trip.
- `glidesId` (string): unique value representing this trip for the given `serviceDate`.
```json
{"serviceDate": "2023-01-20", "glidesId": "ADDED-123"}
```
**Scheduled Trip**
- `serviceDate` (RFC3999 date): the service date for the trip.
- `startLocation` ([Location](#Location)): where the trip is scheduled to start.
- `endLocation` ([Location](#Location)): where the trip is scheduled to end.
- `startTime` ([Time](#Time)): when the trip is scheduled to depart `startLocation`.
- `endTime` ([Time](#Time)): when the trip is scheduled to arrive at `endLocation`.

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

### TripUpdated
This indicates that a trip was updated in some fashion: consist, operators, departure time. The trip is either a scheduled trip, or an added trip that has already appeared in the events stream. Fields which are not present are not considered to be updated.

Fields in the object:
- `type` (string): `"updated"|"added"`. Determines whether this is a TripUpdated object or a [TripAdded](#TripAdded) object. If it's `"added"`, then it's a [TripAdded](#TripAdded) object, see above. Subsequent updates to previously-added trips have `type` `"updated"`.
- `tripKey` ([TripKey](#TripKey)): which trip is being updated.
- `comment` (string, optional): free text information about the trip. Could potentially be the empty string, if a comment was deleted.
- `startLocation` ([Location](#Location) | "unset", optional): the new destination of the train.
- `endLocation` ([Location](#Location) | "unset", optional): the new destination of the train.
- `startTime` ([Time](#Time) | "unset", optional): if present, the new time that the train is expected to depart `startLocation` (or the existing `startLocation` of the trip).
- `endTime` ([Time](#Time) | "unset", optional): if present, the new time that the train is expected to arrive at `endLocation` (or the existing `endLocation` of the trip).
- `cars` (array of [Car](#Car), optional): array of length 1 or 2, containing the car numbers and operators for each car in the train assigned to the trip. If absent, there are no changes to any cars. If present, the length of the array is the length of the train. In a two car train, the front car is listed first.
- `dropped` ([DroppedReason](#DroppedReason) | false, optional): If present the trip was dropped for the provided reason. If set to `false`, the trip is not dropped (and restored if previously dropped).
- `scheduled` ([Scheduled](#Scheduled) | null): For trips in Glides' schedule, this represents the scheduled data for the trip. It will always be present and won't change between subsequent updates to the same trip. For added trips, this will always be `null`.

Some values might be the special string `"unset"`. This means to treat the field as if it had never been modified. This is semantically slightly different than setting the value to the scheduled value. For example, setting `startTime` to `"unset"` implies we don't know when the trip will leave, and the scheduled time might be a good guess, but setting `startTime` to equal the scheduled start time implies that an inspector has confirmed the trip will leave at that time.

*Notes*
- setting a new location or time does not modify the [TripKey](#TripKey) for a scheduled trip.
- If `cars` is updated from a 1-element list to a 2-element list, then Glides SHOULD include data for the 2nd car. This is relevant if a two-car train has a car removed and then re-added. If a field was set before the car was dropped, then when the car is restored, clients MUST assume that the field is `"none"` as opposed to retaining its previous value, but Glides SHOULD include the data to avoid the ambiguity.
- Some fields, such as `startTime` or `cars`, are not relevant to dropped trips. Those fields SHOULD NOT be set in an update that drops a trip or any following updates, until the trip is undropped by setting `dropped` to `false`. However, if a trip is undropped, clients MUST assume all fields retain their values from before the field was undropped. Clients MUST NOT ignore updates to fields when a trip is dropped, even if they aren't relevant for dropped trips. For example, if a trip is dropped, and then the `startTime` is updated, and then it's undropped, the trip's `startTime` is the value set while the trip was dropped. Glides MAY include extra fields in an update that undrops a trip, if they are changed as part of the same event that undropped the trip, or just to remind clients about them now that they're relevant.

## Events

### Editors Changed
Operator sign in and out of editing specific trainsheets. This event publishes changes to who is editing. Since one action by an inspector might result in multiple changes, for example if they sign out of one location and in to another, the changes are published in a list.

In practice, we limit each location to have only one editor, and limit each editor to only one location, but that rule may or may not change and this data in this event doesn't guarantee that.

Note that the author may or may not match the editors in `changes`, since inspectors can change whether other people are editing. For example, when an inspector takes over editing at a location, they may sign out the previous editor. Then they would issue a `stop` change for the previous editor and a `start` change for themself.

Event type: `com.mbta.ctd.glides.editors_changed.v1`
Fields in the event:
- `metadata` ([Metadata](#Metadata)): how the changes were entered in Glides, including the author.
- `changes` (array of [EditorChange](#EditorChange)): a list of start and stop editing events

### Operator Signed In
At the start of their shift, operators need to confirm that they are fit-for-duty and do not have any electronic devices. They currently do this by physically signing a paper trainsheet: in the future, they will do this digitally.

Event type: `com.mbta.ctd.glides.operator_signed_in.v1`
Fields in the event:
- `metadata` ([Metadata](#Metadata)): how the operator was signed in in Glides, including the inspector who signed them in.
- `operator` ([Operator](#Operator)): the operator who signed in
- `signedInAt` (RFC3999 timestamp): the time at which they signed in (separate from the `time` of the event)
- `signature` ([Signature](#Signature)): how the operator confirmed that they signed in.

### Trips Updated
Glides has new information about trips. Consumers who want to know what trains are running should pay attention to this event, and don't need to pay attention to any other events.

Event type: `com.mbta.ctd.glides.trips_updated.v1`
Fields in the event:
- `metadata` ([Metadata](#Metadata)): how the updates were entered in Glides, including the inspector who made the change and the location of the trainsheet.
- `tripUpdates` (array of ([TripUpdated](#TripUpdated)|[TripAdded](#TripAdded))): The list of one or more trips that have new information.

# Reference-level explanation
## Effects on RTR
These inputs are similar to what RTR is already receiving from OCS.
- Trip added: `TSCH NEW`
- Trip updated: `TSCH CON` (set consist), `TSCH DEL` (drop or restore a trip), `TSCH OFF` (change departure time)

One key difference is that the trainsheets do not record the destination or route for trips, so added trips currently only include one end of the trip. RTR SHOULD assume a default pattern for these trips.

## Idempotence

Glides SHOULD produce idempotent events. Consumers SHOULD assume events are idempotent.

If an event is submitted to the stream twice, due to a network error, then the events SHOULD have the same event id, and clients MAY ignore duplicate events, however, they can safely replay events if they occur in short succession.

If an [TripAdded](#TripAdded) object has the same [TripKey](#TripKey) as a previous TripAdded event, it SHOULD have the same data, and SHOULD be ignored. If two [OperatorSignedIn](#OperatorSignedIn) events have the same `operator` and `signedInAt`, then clients SHOULD assume they are the same signature, and the second SHOULD be ignored.

## Short-term storage
Events are only maintained in Kinesis for 24 hours. Some events (such as a [Trips updated][#trips-updated] which modifies the operators) can affect trips more than 24 hours in the future. Clients SHOULD maintain records internally of these future changes. Clients MUST NOT expect that reading the Kinesis stream from the trim horizon will return all events affecting the current service day. Clients MUST tolerate receiving updates for events that reference unseen past events (such as an update to an added trip that the client did not see get added, or a [`EditorsChanged`](#Editors-Changed) event that stops editing for an inspector the client did not see start editing).

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
Consumers MUST ignore fields in events (and in nested objects in events) which they do not understand, trusting that any additional fields are not required to process the event.

## Examples
For all other examples after the first, `specversion`, `source`, `time`, and `id` fields are elided from the top level, and some fields of `data.metadata` are elided.

### Starting Editing
Inspector Alice (badge number: 123) starts her shift at Boston College. The previous inspector (badge number: 456) did not stop editing, so Alice clicks the "Take Over" button in Glides.

```json
{
  "type": "com.mbta.ctd.glides.editors_changed.v1",
  "specversion": "1.0",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-20T09:30:00-05:00",
  "data": {
    "metadata": {
      "author": {
        "emailAddress": "ainspector@example.com",
        "badgeNumber": "123"
      },
      "inputTimestamp": "2023-01-20T09:29:59-05:00",
      "inputType": "take-over-editing"
      "location": {"gtfsId": "place-lake"}
    },
    "changes": [
      {
        "type": "stop",
        "location": {"gtfsId": "place-lake"},
        "editor": {
          "emailAddress": "binspector@example.com",
          "badgeNumber": "456"
        }
      },
      {
        "type": "start",
        "location": {"gtfsId": "place-lake"},
        "editor": {
          "emailAddress": "ainspector@example.com",
          "badgeNumber": "123"
        }
      }
    ]
  }
}
```

### Operator Sign In

Operator Charlie (badge: 789) returns from his break and stops by Inspector Alice's booth. Alice checks him as fit for duty, and he signs in by tapping his badge to the RFID reader attached to Alice's computer.

```json
{
  "type": "com.mbta.ctd.glides.operator_signed_in.v1",
  "data": {
    "metadata": {},
    "operator": {"badgeNumber": "789"},
    "signedInAt": "2023-01-20T09:45:00-05:00",
    "signature": {"type": "tap"}
  }
}
```

### Managing headways
- There are four scheduled trips going to Government Center: 9:55am, 10:00am, 10:05am, and 10:10am
- Due to staffing issues, the 10:05am trip is dropped, along with its return trip.
- Using the "Manage Headways" feature, Alice changes the expected departure times of the remaining trips to 9:56am, 10:02am, and 10:08am.

(Scheduled data is unrealistically abbreviated for this example. Normally there would be information about scheduled operators. See the next example for realistic scheduled operator data.)

```json
{
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "data": {
    "metadata": {
      "inputType": "dropped-trip"
    },
    "tripUpdates": [
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "10:05:00",
          "endTime": "10:52:00"
        },
        "dropped": {"reason": "staffing"},
        "scheduled": {"scheduledCars": [{}]}
      },
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-gover"},
          "endLocation": {"gtfsId": "place-lake"},
          "startTime": "10:55:00",
          "endTime": "10:42:00"
        },
        "dropped": {"reason": "staffing"},
        "scheduled": {"scheduledCars": [{}]}
      }
    ]
  }
}
{
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "data": {
    "metadata": {
      "inputType": "manage-headways"
    },
    "tripUpdates": [
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "09:55:00",
          "endTime": "10:42:00"
        },
        "startTime": "9:56:00",
        "scheduled": {"scheduledCars": [{}]}
      },
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "10:00:00",
          "endTime": "10:47:00"
        },
        "startTime": "10:02:00",
        "scheduled": {"scheduledCars": [{}]}
      },
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "10:10:00",
          "endTime": "10:57:00"
        },
        "startTime": "10:08:00",
        "scheduled": {"scheduledCars": [{}]}
      }
    ]
  }
}
```

### Splitting a 2-car train into two 1-car trips
- There are two scheduled trips: 9:55am and 10:00am
- Operator A (badge number: 456) and Operator B (badge number: 567) are scheduled to perform the 9:55am
- Only 1 trainset (3800 and 3850) is available
- In order to maintain headways, Inspector Alice will split the trainset
- Operator A will take 3800 and run the 9:55am
- Operator B will take 3850 and run the 10:00am
- Alice does this by dropping the second trip and adding a new one-car trip (and also its return). In real life she would probably assign B to the 2nd trip, and all the trips would be round trips. The example does it this way to provide an example of adding a trip and an example of how trips in the stream might not align to GTFS trips.

```json
{
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "data": {
    "metadata": {
      "inputType": "dropped-trip"
    },
    "tripUpdates": [
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "10:00:00",
          "endTime": "10:47:00"
        },
        "dropped": {"reason": "ran as single"},
        "scheduled": {
          "scheduledCars": [
            {
              "run": "506",
              "operator": {"badgeNumber": "678"}
            },
            {
              "run": "507",
              "operator": {"badgeNumber": "789"}
            }
          ]
        }
      }
    ]
  }
}
{
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "data": {
    "metadata": {
      "inputType": "edit-trip"
    },
    "tripUpdates": [
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "startLocation": {"gtfsId": "place-lake"},
          "endLocation": {"gtfsId": "place-gover"},
          "startTime": "09:55:00",
          "endTime": "10:42:00"
        },
        "comment": "single",
        "cars": [
          {
            "label": "3800",
            "operator": {"badgeNumber": "456"}
          }
          /* note that there is no second car, this is how the second car is removed */
        ],
        "scheduled": {
          "scheduledCars": [
            {
              "run": "504",
              "operator": {"badgeNumber": "456"}
            },
            {
              "run": "505",
              "operator": {"badgeNumber": "567"}
            }
          ]
        }
      }
    ]
  }
}
{
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "data": {
    "metadata": {
      "inputType": "add-trip"
    },
    "tripUpdates": [
      {
        "type": "added",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "glidesId": "ADDED-1"
        },
        "startLocation": {"gtfsId": "place-lake"},
        "startTime": "10:00:00",
        "cars": [
          {
            "label": "3850",
            "operator": {"badgeNumber": "567"}
          }
        ],
        "scheduled": null
      },
      {
        "type": "added",
        "tripKey": {
          "serviceDate": "2022-01-20",
          "glidesId": "ADDED-2"
        },
        "endLocation": {"gtfsId": "place-lake"},
        "previousTripKey": {
          "serviceDate": "2022-01-20",
          "glidesId": "ADDED-1"
        },
        "cars": [
          {
            "label": "3850",
            "operator": {"badgeNumber": "567"}
          }
        ],
        "scheduled": null
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
  "type": "com.mbta.ctd.glides.trips_updated.v1",
  "time": "2023-01-23T01:25:00-05:00",
  "data": {
    "metadata": {
      "inputType": "edit-trip"
    },
    "tripUpdates": [
      {
        "type": "updated",
        "tripKey": {
          "serviceDate": "2023-01-22",
          "startLocation": {"gtfsId": "place-matt"},
          "endLocation": {"gtfsId": "place-ashmt"},
          "startTime": "25:30:00",
          "endTime": "25:38:00"
        },
        "location": {"gtfsId": "place-matt"},
        "startTime": "25:45:00",
        "scheduled": {"scheduledCars": [{}]}
      }
    ]
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

## Alternative: use inputTimestamp as the event time
As proposed, the event's `time` field is the time that the Glides server received the input. CloudEvents [`suggests`](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md#time) using "when the occurence happened", which would be when the input was made, i.e. what this proposal puts in the [`metadata.inputTimestamp`](#Metadata) field. The Glides server time is used instead because:

- The `inputTimestamp` is generated by the client device, and so is in some sense less trustworthy. What if a client's clock is set wrong? We don't do any validation, and we can't, because we can't tell the difference between an incorrect time and a correct time that was delayed by network issues. This isn't so much a specific concern as a general practice of not depending on non-validated client data.
- Glides applies updates in the order they're received. If the received timestamp is used as the event time, then applying events in order by their timestamp will give you the same state as in Glides. If we use the client timestamp, then if an event is delayed so it's received after another later input, a client who checks the event timestamp could apply them in the wrong order and end up with a different state than Glides. This could be mitigated by documenting that a client-provided event timestamp is unreliable, but that's easier to document if the client timestamp is included with other input metadata instead.
- The CloudEvents docs seem to say the event time's consistency is more important than it being the original occurence time. It'll be much easier to stay consistent with server-generated timestamps.

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

## Alternative: More specific trip update events
Earlier drafts of this RFC included some events for more specific kinds of trip updates, such as a "trip-dropped" event or a "trip-departed" event. It was even considered to use separate events for each updatable field.

More events and more detailed events would have allowed more precise data to be sent in some cases, and would have better reflected the actions that inspectors take in Glides. For example, a specific "trip-dropped" event could prevent confusing data like setting a `startTime` in the same event as dropping a trip, and a "manage-headway" event could better portray the inspector's intent than a list of generic `TripUpdated` objects with new times.

It was instead decided that all trip updates should result in the same generic `TripsUpdated` event, with a list of `TripUpdated` objects that don't reference the action the inspector took, or the location of the inspector.

Reasons:
- Clients only have to look at the data in one event in order to get all updates that affect passenger service.
- The event (and clients) can focus on new information about service, instead of on what happened within Glides.
- If Glides adds new ways for inspectors to enter data, it can produce the same event with the same data format, instead of requiring backwards-incompatible new events.

The `inputType` field in [Metadata](#Metadata) was added as an escape hatch out of the abstraction, so that clients who want to can see the implementation detail of how the inspector created the update in Glides.

## Alternative: Separate trip-added event
The current RFC has added trips as a modified case of a TripUpdated object in the "trips-updated" event. Previous versions of the RFC had a separate "trip-added" event.

Advantages to separate Added and Updated events:
- Clients probably need to handle added and updated trips slightly differently.
- Combining them creates more complicated cases and conditional requirements in the already-complicated TripsUpdated event.

Advantages to combining them:
- Clients only have to look at the data in one event in order to get all updates that affect passenger service.
- Better represents how TripAdded can contain basically the same information that TripUpdated can.
- The TripsUpdated event can contain a list of objects, instead of needing a separate Batched event to combine arbitrary events. (The Batched event is not otherwise needed, because the other operator_signed_in and editors_changed events never happen at the same time as trips_updated.)

## Alternative: Include snapshot of all data, not just changes
In order to know everything about a trip, a client must be listening and remember all past changes. Glides could publish everything it knows about a trip so far, alongside updates or as an independent event.

Benefits:
- If a change was made more than 24hr ago, and is not in Kinesis anymore, a snapshot lets clients see the data even if they missed the old event.
- Reduces the chance that clients and Glides get out of sync if their logic for updating state based on changes is not the same.
- Some clients may be able to ignore all changes and only look at the snapshot, simplifying their implementation.

Disadvantages:
- More data, more complicated schema, more effort.
- Might not completely solve the out-of-date data problem for some edge cases.

Because this is just a hypothetical benefit, and adding a snapshot in later would be backwards-compatible, it won't be included unless there's a concrete reason for it.

## Alternative: Split "cars" field into separate fields.
Car numbers and operators are joined into a `cars` field with a list of objects. Previous versions of the RFC had them as separate fields, each with their own list.

Similarly, there is a `scheduled.scheduledCars` list that joins scheduled runs and operators together.

Disadvantages of separate lists:
- Requires keeping the lists the same length.
- Requires zipping the lists together to get all data about each car.
- Requires a special "unmodified" value for elements of the list that haven't been modified. A joined `cars` list can use the standard way to show a datum has not been modified by omitting it from the object.

# Prior art
[Trike](https://github.com/mbta/trike) uses a similar approach to provide events to RTR for influencing predictions and to OCS Saver for future processing by LAMP / OPMI. These events come from the heavy-rail dispatching system, which uses another implementation of trainsheets. This approach is described in [RFC4](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0004-socket-proxy-ocs-cloudevents.md) and [RFC5](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0005-kinesis-proxy-json.md).

# Unresolved questions
- While CloudEvent formats are included here, the final specific format should be settled on as a conversation between the Glides, Transit Data, and LAMP teams.

# Future possibilities
- As Glides becomes more widely adopted, further information entered by Chief Inspectors and other officials can be included as events, such as short turns, holds, express trips, non-revenue trips, and yard management. However, this is not currently scheduled for implementation.
