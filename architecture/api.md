# V3 API design

## Context/scope

As we started re-writing the website, we wanted the data to be based on the existing GTFS and GTFS-RT data. We also hoped to provide this data as an API to the public. The team is 3 developers.

## Goals / non-goals

- provide an interface that makes it easy to build the new website
- separation of more static data (routes, schedules) from more dynamic data (vehicle locations + predictions) for caching
- short response times
- very scalable
- consume GTFS/GTFS-RT with little customization
- (non-goal) support clients fetching the full set of GTFS/GTFS-RT data

## Design

- Languge: Elixir. soft-realtime is a good match for an API, Ruby-like syntax, easy concurrency, functional
- Web framework: Phoenix. Similar MVC structure as Django or Rails.
- Data: Mnesia + [R*][erl-rstar]. Can keep all the data in memory for scalability and performance, without a dependency on another server.
- API format: [JSON:API][json-api]. It's standardized, and supports a lot of nice features (filters, including additional data, sorting).

## Alternatives considered

- Python: fun to write and can be functional, but not as good at concurrency due to the GIL.
- Django: has a lot built-in (including an admin), but it's a big framework with a lot we wouldn't need.
- Postgres/PostGIS: standard query language with GIS query support, but potentially slower imports and another server to support
- Continue using the V2 API: doesn't appear to be performant enough for the amount of traffic we expected.

Elixir is a newer language, so it's less well-known than Python or Ruby. Mnesia has a much different query language, and has a slower startup than if the data were already present in a separate database. JSON:API can be more verbose than a custom JSON interface, and none of the developers were familiar with GraphQL.

## Cross-cutting concerns (security, privacy, scalability, observability)

- Only consumes our public GTFS/GTFS-RT feeds
- Logs are sent to Splunk, including ehmon metrics
- Can log the incoming GTFS/GTFS-RT data to S3 if required

[erl-rstar]: https://github.com/armon/erl-rstar
[json-api]: https://jsonapi.org/
