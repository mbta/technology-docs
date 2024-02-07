- Feature Name: Telemetry

- Start Date: 2024-01-24

- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)

- Asana task: [asana link](https://app.asana.com/)

- Status: Proposed

# Summary

Observability is the ability to measure the internal states of a system by examining its outputs.<sup>[1](https://www.splunk.com/en_us/blog/learn/observability.html)</sup>
It's what allows us to understand how our applications and systems are working.
There are three pillars to observability:<sup>[2](https://www.crowdstrike.com/cybersecurity-101/observability/three-pillars-of-observability/)</sup>

  1. **Logs** are the archival or historical records of system events and errors, which can be plain text, binary, or structured with metadata.
  2. **Metrics** are numerical measurements of system performance and behavior, such as CPU usage, response time, or error rate.
  3. **Traces** are the representations of individual requests or transactions that flow through a system, which can help identify bottlenecks, dependencies, and root causes of issues.

TID engineers currently make extensive use of logging.
But, we don't currently have an established mechanism for collecting metrics or traces.
This RFC proposes multiple pathways for collecting metrics in our current observability platform Splunk.

# Motivation

Logs are incredibly useful for debugging and other unstructured research like a security audit.
But, as noted above, they are just one of three tools used to observe systems.

> It might be tempting to think that logs can solve every use case. As the amount of data grows, however, a logs-only solution will become costly and relatively slow for a small set of regular searches, usually connected to alerts. This is because the process by which logs must be categorized and batched takes much more time and is much more computationally intensive than the metrics process...<sup>[3](https://www.splunk.com/en_us/blog/learn/logs-vs-metrics.html)</sup>

Metrics are stored as a timeseries which means they are optimized for storage and retrieval as numbers in time.
This is what makes them cheap to store and fast to retrieve.
TID developers have already noted that many queries are incredibly slow if not impossible to run.
By converting those observations to metrics, we can speed up said queries, and in many cases make alerts that were previously impossible possible.

# Guide-level explanation

This proposal is primarily concerned with capturing and transmitting both built-in and custom metrics.

What is a metric and how is it different from a log? Let's define a log as a structured **event** detailing information about a program's execution. A metric, on the other hand, is a single **number** useful for informing us about a program's execution. They are useful in different ways.

For example, let's say you want to know how many requests your application served per minute over the last hour. Using logs, you would send a log to Splunk for every single request. Dotcom regularly receives 150 requests per second or 9,000 requests per minute. So, if you wanted to look at the span of an hour, you would query 540,000 logs and bin each log by its timestamp. Lastly, you would count each log in each bin. You can do this in Splunk, but it is slow. And, it gets slower and slower as you increase the time horizon or amount of data. This is because logs are stored as documents optimized to be searched rather than calculated upon.

If you used metrics instead, your application would keep an internal count of the number of requests it was receiving. Every minute (this is tuneable per application) it would emit that number to Splunk along with a timestamp and reset itself. So, in a minute, it would emit the number 150 once. That is 1 request instead of the 9,000 above. Reviewing an hour of data would entail nothing more than looking at 60 numbers. This is clearly much faster than querying, sorting, and counting.

Elixir libraries such as Phoenix, Ecto, Nebulex, and many others have adopted the [telemetry](https://hexdocs.pm/telemetry/readme.html) library as a standardized way to give users observability into their performance. It emits events which can be turned into metrics automatically via [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html).

Providing custom metrics is as simple as logging:
```elixir
:telemetry.execute([:web, :request, :done], %{latency: latency}, %{request_path: path, status_code: status})
```

Once your application is emitting metrics, you need to report on them. Multiple reporter libraries exist such as [statsd](https://github.com/beam-telemetry/telemetry_metrics_statsd) or [CloudWatch](https://github.com/bmuller/telemetry_metrics_cloudwatch). You can also write your own such as this [custom reporter from dotcom](https://github.com/mbta/dotcom/blob/master/lib/cms/telemetry/reporter.ex).

See More: [Introduction to Telemetry in Elixir](https://blog.miguelcoba.com/introduction-to-telemetry-in-elixir).

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

There are really two phases to discuss: getting metrics out of our applications and getting those metrics into Splunk.

### Getting metrics out of our applications

For getting metrics out of our Elixir applications, we want to use the [telemetry](https://hexdocs.pm/telemetry/readme.html) library along with [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html) to convert them to metrics and the [telemetry_poller](https://hexdocs.pm/telemetry_poller/readme.html). The telemetry library standardizes a method for emitting events. The telemetry metrics library turns those events into metrics, and the telemetry poller gathers those metrics and emits them to a reporter at whatever intervals you define. You can see an [example implementation from dotcom](https://github.com/mbta/dotcom/blob/master/lib/cms/telemetry.ex).

### Getting metrics into Splunk

There are three ways to get metrics into Splunk, and teams should feel free to pursue the path that best fits their application and expertise.

#### Splunk Universal Forwarder

The UF route takes the most work upfront. But, it is the most flexible. Any service can emit statsd and it will make its way to Splunk. If we decide to move off of Splunk, we just need to find a provider that supports statsd. Furthermore, the statsd layer gives us a way to observe metrics while developing locally.

  - Would be run as a sidecar
  - Can use statsd so any app in any language would just need to emit statsd
  - There are also tons of statsd libraries
  - Statsd is a format, so if we wanted to swap out Splunk, we could
  - There is a statsd telemetry reporter already https://github.com/beam-telemetry/telemetry_metrics_statsd
  - We can run a statsd server locally and inspect metrics while deving
  - DOWNSIDE: the UF is finicky so we would have to figure out how to configure it in a container
  - DOWNSIDE: would entail making infra changes

### Splunk HTTP Event Collector

For this project, the HEC seems like the next best option. It will be the least work, but that is because it is the least flexible. We would have monitor metrics in Splunk, but we wouldn’t get any reuse. The metrics being emitted from the dotcom application still wouldn’t be picked up. To do that, we would have to write an Elixir library.

  - The monitor could easily just send JSON requests
  - DOWNSIDE: no telemetry reporter seems to exist so we would have to write one for elixir apps

### AWS Cloudwatch

The AWS CloudWatch custom metrics would also be a good option for the larger organization. The biggest downside is that we would be paying both AWS and Splunk for every metric. If we shift focus from logs to metrics, this could be a non-trivial amount of money. The other downside is that it locks us into CloudWatch.

  - There is a telemetry reporter for cloudwatch https://github.com/bmuller/telemetry_metrics_cloudwatch
  - The monitor uses node and the AWS sdk supports CloudWatch metrics
  - DOWNSIDE: we would be paying for metrics in CloudWatch and Splunk
  - DOWNSIDE: ties us to CloudWatch

# Drawbacks

[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- Can we learn something about this proposal from other projects in CTD?
- Can we learn something about this proposal from other transit agencies?
- Can we learn something about this proposal from past experiences in other jobs or projects?

This section is intended to encourage you as an author to think about the lessons from other places, and provide readers of your RFC with a fuller picture.

If there is no prior art, that is fine.

Note that while precedent is some motivation, it does not on its own motivate an RFC.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities

[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project in your proposal. Also consider how this all fits into the roadmap for the project.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
