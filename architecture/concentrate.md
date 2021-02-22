# Concentrate design

## Context/scope

The MBTA has real-time information for many modes, coming from different sources:
- Subway and light rail come from RTR (formerly HRRT)
- Commuter rail comes from Keolis, through Commuter Rail boarding and TrainLoc
- Bus comes from Swiftly through TransitMaster, Samsara, and Busloc

We need a single feed for the API and other GTFS-RT consumers.

## Goals / non-goals

- Process all of our GTFS-RT sources into a single output feed
- Low latency between fetching new data and providing it to the output feeds
- Cheaper than the existing solution
- Ensure that GTFS-RT outputs meet the GTFS-RT standard
- Respond to service alerts by making simple changes to the output
- Little/no MBTA-specific logic
- Resilient to failures within a short period of time
- (non-goal) high-availability
- (non-goal) horizontal scalability

## Design

- Language: Elixir
- Deployment: ECS on Fargate
- Upstream: GTFS-RT/GTFS providers (both internal to MBTA and vendors)
- Downstream: V3 API, large third-parties (Google, Apple, Transit, &c) which consume GTFS-RT

Internally, Concentrate is a GenStage (github.com/elixir-lang/gen_stage) pipeline.  Each GTFS-RT / GTFS source is a producer, and each GTFS-RT output, Reporter, and Sink is a consumer. GenStage manages backpressure to ensure that we aren't overloading any part of the system.

Concentrate provides no public API: it uploads GTFS-RT files to an S3 bucket for the API and other consumers to use.

Configuration is stored in environment variables with the ECS tasks, and all inputs are loaded from S3 at startup: there is no need for a database to maintain state across deployments/crashes.

Instead of running multiple instances, we rely on ECS/Fargate to ensure that there's always an instance running.

## Constraints

Some of our data sources (Swiftly) have a rate limit that's close to the rate that we want a single instance to fetch. This prevents us from running multiple Concentrate instances to provide high-availability. However, we do run multiple instances for a short time during deploys.

## Alternatives considered

The existing implementation was MBTA-realtime, provided by IBI. This ran on a large Windows server with a SQL Server database, and cost thousands of dollars.

## Cross-cutting concerns (security, privacy, scalability, observability)

- Credentials such as API keys are stored in AWS Secrets Manager (now: originally they were directly in the configuration)
- All data is parsed out of its original format, so additional private data is ignored
- Elixir logs are sent to Splunk, including ehmon metrics
- Reporter interface provides a way to log statistics and other relevant information about processing
- Sink interface logs the fetched files to a local filesystem or an S3 bucket
- Issues requiring intervention are sent to PagerDuty
