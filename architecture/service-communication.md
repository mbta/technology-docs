# Service Communication design

## Context/scope

As we grow the sources of real-time data internally at CTD, we need a standard way for them to communicate that data out.

## Goals / non-goals

- Easy for developers to implement
- Support additional data besides what's in the GTFS-RT Protobuf definition
- Support medium-frequency (1/s) updates

## Design

GTFS-RT Enhanced is a JSON feed in the same shape as the GTFS-RT Protobuf files.
```
{
  "header": {...},
  "entity": [{...}, {...}]
}
```
Each object can have additional data in any of the objects, to represent additional data as required by the application.
- RTR: stops away, revenue service
- Busloc: operator information
- Commuter Rail: track boarding status

By default, the additional fields are private, and do not pass through Concentrate into the public feeds. However, support can be added to Concentrate to include them.

The files themselves are uploaded to S3, at whatever frequency is appropriate for the application.

## Constraints

S3 is only eventually consistent, so it's possible for clients to read a stale version of the file. Clients should use the If-Modified-Since header when fetching to ensure they always get the latest version.

Raw JSON files are much larger than Protobuf files. To ensure that we can efficiently transport them, uploads and downloads should be GZip compressed.

## Alternatives considered

- A queue could have been more efficient, but would have had much larger operational overhead which we didn't have the staff to support.
- At the time, the GTFS-RT Protobuf spec didn't have a space for private extensions. Even so, we'd need to maintain a fork of it to support the additional fields across our applications.

## Cross-cutting concerns
- Data is secured in S3 by using S3 bucket policies and IAM roles
