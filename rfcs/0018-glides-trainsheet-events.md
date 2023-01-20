- Feature Name: `glides-trainsheet-events`
- Start Date: 2023-01-20
- RFC PR: [mbta/technology-docs#18](https://github.com/mbta/technology-docs/pull/18)
- Status: Proposed

# Summary
Glides will provide high-level trainsheet information as CloudEvents in a Kinesis stream

# Motivation
As part of digitizing the currently paper-based trainsheets that define operational conditions on the Green Line, Glides will be gathering information about consist assignments, dropped/added trips, and departure time adjustments from its users: pull-out inspectors at selected terminal stations. This data can serve to improve the rider-facing predictions produced by RTR, especially for departure times at a terminals, which currently do not have departure predictions at all.

# Guide-level explanation
The Glides trainsheet functionality hews closely to what the paper-based trainsheets provide.

- sign an operator in, marking them fit-for-duty
- add a trip (1-way from the current terminal, 1-way arriving at the current terminal, or a round-trip from the current terminal)
- drop a trip
- add/change/remove train cars to a trip
- add/change/remove the operator assigned to a trip
- set a new departure time for a trip

These actions are always taken by a logged-in user (generally an inspector).

## CloudEvents
All events will be in the [CloudEvents v1.0.2](https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md) format (or later), using the JSON encoding. The event types will be under the `com.mbta.ctd.glides` event namespace, and will be in the past-tense (i.e. `trip-added` rather than `add-trip`).

If multiple events happen as a part of the same user action, they can be written as a Batch event, wrapping all of the events in a JSON array.

## Kinesis
Events will be written to a Kinesis stream. Each Glides environment (`dev`, `dev-green`, and `prod`) will have a separate stream for their events, named `ctd-glides-<environment>`. The partition key will be a hash of the station at which the inspector is working and the inspector's identity, which ensures that multiple events from a single inspector are ordered correctly if the messages are distributed across multiple shards. As the stream will include potential PII (operator badge information) it should be configured as encrypted-at-rest.

## Value types

### Author
Each message will include an `author` key, indicating who performed the given intervention.

- `username` (string): the username of the logged-in user
- `badgeNumber` (string, optional): the badge number of the logged-in user
- `location` (Location, optional): where the logged-in user is located
```json
{
  "username": "example",
  "badgeNumber": "123456",
  "location": "gtfs:place-lake"
}

or

{
  "username": "pswartz"
}
```
It is represented as an object to provide future extensibility if needed.

### Car
Cars (individual parts of a train consist) will be described by their 4 digit number.
```json
{
  "carNumber": "3800"
}
```

### Location
One of:
- `gtfs:<GTFS stop ID>`, where the GTFS `location_type` is 0 (stop) or 1 (station)
- `id:<other>`: Other unique identifier for non-revenue locations

### Operator
Operators will be described by their badge number:
```json
{
  "badgeNumber": "123"
}
```

### Optional T
A value that can either be an instance of T, or `null`. For example, Optional Car can be
```json
{ "carNumber": "3800"}
# or 
null
```

### Time
A [GTFS Time](https://github.com/google/transit/blob/master/gtfs/spec/en/reference.md#field-types).
> Time in the HH:MM:SS format (H:MM:SS is also accepted). The time is measured from "noon minus 12h" of the service day (effectively midnight except for days on which daylight savings time changes occur). For times occurring after midnight, enter the time as a value greater than 24:00:00 in HH:MM:SS local time for the day on which the trip schedule begins.  
> _Example: `14:30:00` for 2:30PM or `25:35:00` for 1:35AM on the next day._

### Trip Key
For scheduled trips, Glides does not have the same trip ID used by RTR. For added trips, while an ID is provided, it can be the case that an added trip from the Glides perspective is matched to a GTFS trip, in which case RTR will use the GTFS trip ID in publishing the information.

In order to reduce message duplication, a Trip Key is used to identify both added and scheduled trips.

**Added Trip**
- `serviceDate` (RFC3999 date): the service date for the trip
- `id` (string): unique value representing this trip for the given `serviceDate`
```json
`{"serviceDate": "2023-01-20", "id": "ADDED-123"}
```
**Scheduled Trip (departure time from origin)
- `serviceDate` (RFC3999 date): the service date for the trip
- `departureTime` (Time): when the vehicle is scheduled to depart the given location
- `startLocation` (Location): where the trip is departing from
```json
`{"serviceDate": "2023-01-20", "departureTime": "25:01:02", "startLocation": "gtfs:place-lake"}
```
**Scheduled Trip (arrival time at destination)
- `serviceDate` (RFC3999 date): the service date for the trip
- `arrivalTime` (Time): when the vehicle is scheduled to arrive at the given location
- `endLocation` (Location): where the trip is ending
```json
`{"serviceDate": "2023-01-20", "arrivalTime": "09:10:11", "endLocation": "id:yard-inner"}
```
**Scheduled Trip (known GTFS trip ID)**
This should only be used in the case where Glides is frequently fetching an updated GTFS file (at least hourly).

- `serviceDate` (RFC3999 date): the service date for the trip
- `gtfsId` (string): GTFS trip ID
```json
`{"serviceDate": "2023-01-20", "gtfsId": "123456790"}
```

### Trip Type
When adding a trip, there are three kinds of trips which can be added:
- `"arrival"`: a trip arriving to `endLocation` at `time`
- `"departure"`: a trip departing from `startLocation` at `time`

## Events
### Operator signed in
At the start of their shift, operators need to confirm that they are fit-for-duty and do not have any electronic devices. They do this by either physically signing a paper trainsheet, or by typing their badge number into Glides. This event represent either type of signin, but may not be generated in the case where an operator only signs into the paper trainsheet.

Event type: `com.mbta.ctd.glides.operator-signed-in`
Fields in the event:
- `author` (Author): the inspector who signed in the operator
- `operator` (Operator): the operator who signed in
- `signedInAt` (RFC3999 timestamp): the time at which they signed in (separate from the `time` of the event)

### Trip added
This creates a new trip in the timesheet. It may or may not be a new trip relative to GTFS (see the splitting a train reference example).

Event type: `com.mbta.ctd.glides.trip-added`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): provides a unique identifier for the newly created trip
- `route` (string, optional): if specified, the route tag for the trip. This is different from a GTFS route ID.
- `startLocation` (Location, conditionally required): where the trip will be starting. Required if `tripType` is `"departure"`.
- `endLocation` (Location, conditionally required): where the trip will be ending. Required if `tripType` is `"arrival"`.
- `time` (Time, conditionally required): when the added trip is expected to depart (`tripType` of `"departure"`) or arrive (`tripType` of `"arrival"`). Required when `tripType` is `"departure"` or if `previousTripKey` is not specified.
- `tripType` (TripType): whether `time` for the new trip is the when the trip is departing `startLocation` (`"departure"`) or when the trip is arriving at `endLocation` (`"arrival"`).
- `previousTripKey` (Trip Key, optional): links the newly created trip to the trip immediately before it. If the Trip Key refers to an added trip, it may not have been seen in the event stream at the time of this message.
- `nextTripKey` (Trip Key, optional): links the newly created trip to the trip after it. If the Trip Key refers to an added trip, it may not have been see in the event stream at the time of this message.
- `consist` (array of Optional Car, optional, default: empty list): the cars assigned to perform this trip. If a car is `null`, then no car is currently assigned to that position in the consist.
- `operators` (list of Optional Operator, optional, default: empty list): the array of operators assigned to perform this trip. If an operator is `null`, then no operator is assigned to the car in that position in the consist.
- `comment` (string, optional): free text with additional information about the trip

*Notes*
- The event should only include values which were explicitly included by the `author`: inferred values should not be included.
- `consist` and `operators` MUST be the same length; if they are not, the behavior of consumers is undefined.

### Trip dropped
This removes a trip (either added or scheduled), indicating that it will not be running.

Event type: `com.mbta.ctd.glides.trip-dropped`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): which trip to drop. can be either an added or scheduled trip.
- `reason` (string): a text description of why the trip is being dropped

### Trip restored
This restores a trip (either added or scheduled) that was previously dropped.

Event type: `com.mbta.ctd.glides.trip-restored`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): which trip to drop. can be either an added or scheduled trip.

### Trip updated
This indicates that a trip was updated in some fashion:  consist, operators, departure time. Fields which are not present are not considered to be updated.

Event type: `com.mbta.ctd.glides.trip-updated`
Fields in the event:
- `author` (Author): the inspector adding the trip
- `tripKey` (Trip Key): which trip is being updated
- `comment` (string, optional): free text information about the trip
- `route` (string, optional): if specified, the new route tag for the trip. This is different from a GTFS route ID.
- `location` (Location, optional): where the time/consist is being updated for. This can be different from the location in the `author` value, if the author is updating information about the train relative to a different location.
- `arrivalTime` (Time, optional): if present, the new time that the train is expected to arrive at the given location. If `location` is not specified, this is the time the vehicle is arriving at the final location of the trip.
- `departureTime` (Time, optional): if present, the new time that the train is expected to depart the given location. If `location` is not specified, this is the time the vehicle is departing the first location of the trip.
- `consist` (array of Optional Car, optional): the cars assigned to perform this trip. If a car is `null`, then no car is currently assigned to that position in the consist. The front car should be listed first.
- `operators` (array of Optional Operator, optional): the list of operators assigned to perform this trip. If an operator is `null`, then no operator is assigned to the car in that position in the consist.

*Notes*
- `consist` and `operators` MUST be the same length; if they are not, the behavior of consumers is undefined.

# Reference-level explanation
## Effects on RTR
These inputs should be similar to what RTR is already receiving from OCS.
- Trip added: `TSCH NEW`
- Trip dropped: `TSCH DEL` with delete status 1
- Trip restored: `TSCH DEL` with delete status 0
- Trip updated: `TSCH CON` (set consist), `TSCH OFF` (change departure time)

One key difference is that the trainsheets do not record the destination for trips, so added trips only include one stop. RTR should assume a default pattern for these trips.

## Long-term storage / querying
For future use, events will be archived to S3, using Kinesis Firehose. They can either be sent to LAMP's folder, or a new folder. As the stream contains potential PII (operator badge information) any long-term storage (S3, database) must be encrypted at rest.

### Schema
All events should have their schema documented in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository.

### Compatibility
Changes to the events can happen in one of two ways:
- backwards-compatible changes (such as adding a new field with a default value) where old consumers can still interpret new events can be made while keeping the same event type. 
- incompatible-changes require creating a new event type. Some options as described in [RFC 4](https://github.com/mbta/technology-docs/blob/main/rfcs/accepted/0004-socket-proxy-ocs-cloudevents.md) are `<type>.<date>` (adding the date of the new event) or `<type>.v2` (creating a new version). Each version should be documented separately in the [mbta/schemas](https://github.com/mbta/schemas) GitHub repository.

Consumers must ignore event types that they do not understand.

Updating the RFC is not necessary, but the schema repository must be kept up to date if the events are changed.

## Examples
### Train departing 15 minutes later than scheduled
- Inspector Alice (badge number: 123) is working at Mattapan
- The 1:30am trip is being held 15 minutes to wait for a final Red Line train
```json
{
  "specversion": "1.0",
  "type": "com.mbta.ctd.glides.trip-updated",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-23T01:25:00-05:00",
  "data": {
    "author": {
      "username": "ainspector",
      "badgeNumber": "123",
      "location": "gtfs:place-matt"
    },
    "tripKey": {
      "serviceDate": "2023-01-22",
      "startLocation": "gtfs:place-matt",
      "departureTime": "25:30:00"
    },
    "departureTime": "25:45:00"
  }
}
```

Note that the date of the the event is 2023-01-23, but the service date is 2023-01-22. The hour in the time itself is greater than 24 hours, to reflect that the time is the next calendar day.
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
  "type": "com.mbta.ctd.glides.operator-signed-in",
  "source": "glides.mbta.com",
  "id": "19fdb184-7dd6-4664-8472-04bd6177ec44",
  "time": "2023-01-20T09:45:10-05:00",
  "data": {
    "author": {
      "username": "ainspector",
      "badgeNumber": "123",
      "location": "gtfs:place-lake"
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
  "type": "com.mbta.ctd.glides.operator-signed-in",
  "data": {
    "operator": {
      "badgeNumber": "567"
    },
    "signedInAt": "2023-01-20T09:45:00-05:00"
  }
}
{
  "type": "com.mbta.ctd.glides.trip-dropped",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
      "startLocation": "gtfs:place-lake",
      "departureTime": "10:00:00"
    },
    "reason": "ran as single"
  }
}
{
  "type": "com.mbta.ctd.glides.trip-updated",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
      "startLocation": "gtfs:place-lake",
      "departureTime": "09:55:00"
    },
    "operators": [
      {"badgeNumber": "456"}
    ],
    "consist": [
      {"carNumber": "3800"}
    ],
    "comment": "single"
  }
}
{
  "type": "com.mbta.ctd.glides.trip-added",
  "time": "2023-01-20T09:50:15-05:00",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
      "id": "ADDED-1"
    },
    "startLocation": "gtfs:place-lake",
    "departureTime": "10:00:00",
    "tripType": "departure",
    "nextTripKey": {
      "serviceDate": "2022-01-20",
      "id": "ADDED-2"
    },
    "consist": [
      {"carNumber": "3850"}
    ],
    "operators": [
      {"badgeNumber": "567"}
    ],
    "comment": "single"
  }
}
{
  "type": "com.mbta.ctd.glides.trip-added",
  "time": "2023-01-20T09:50:15-05:00",
  "data": {
    "tripKey": {
      "serviceDate": "2022-01-20",
      "id": "ADDED-2"
    },
    "endLocation": "gtfs:place-lake",
    "tripType": "arrival",
    "previousTripKey": {
      "serviceDate": "2022-01-20",
      "id": "ADDED-1"
    },
    "consist": [
      {"carNumber": "3850"}
    ],
    "operators": [
      {"badgeNumber": "567"}
    ],
    "comment": "single"
  }
}

```

In this instance, the inspector dropped the 10:00am trip, and created a new trip with the same departure time. Ideally, RTR will treat this the same as if the 10:00am were updated to be a 1-car trip, but inspectors do not always enter the data in that fashion.

# Drawbacks
- This requires Glides to include new functionality to write their messages into a Kinesis stream. However, they already use `ex_aws` so additional messages should not be hard to add.
 
# Rationale and alternatives
- RTR already consumes a Kinesis stream of CloudEvents from OCS, 
- so it's a limited additional lift to consume a new Kinesis stream
- Kinesis Firehose allows writing the stream of messages to S3 for LAMP to consume, without needing a separate integration

## Alternative: split update types into different events
Examples:
- set departure time
- set/unset operator
- set/unset consist

The current trainsheet interface does not distinguish between these cases: all changes are saved at the same time. Functionality to separate them would need to be added to Glides, or multiple events would need to be written in a batch, increasing duplication.

## Alternative: departure time as RFC3999 string
This would be easy to parse with standard tools, but doesn't match existing OCS behavior which only provides a time.

## Alternative: `dropped` boolean in Trip updated
While this does eliminate the need for the Trip dropped and Trip restored messages, it also allows for confusing Trip updated messages. How should clients interpret a Trip updated message which included `dropped` True, along with other changes to the trip?

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
# Unresolved questions
- While some sample CloudEvent formats are included here, the specific format should be settled on as a conversation between the Glides and Transit Data teams.
- The UI does not currently give a `reason` for edits, only for dropped trips. Should there be a `reason` for edits (separate from `comment`)?
- Some trainsheet entries (removing an operator from their trips) can happen the day before. Do we need to persist any events longer than the 24 hour Kinesis default?
- If Glides starts being used at non-terminal, non-yard stations, trip departure times may not be unique (Government Center)

# Future possibilities
- As Glides becomes more widely adopted, further information entered by Chief Inspectors and other officials can be included as events, such as short turns, express trips, and non-revenue trips. However, this is not currently scheduled for implementation.
