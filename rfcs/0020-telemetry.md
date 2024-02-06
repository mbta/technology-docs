- Feature Name: Telemetry

- Start Date: 2024-01-24

- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)

- Asana task: [asana link](https://app.asana.com/)

- Status: Proposed

# Summary

TID engineers currently make extensive use of logging for a wide range of use cases, primarily from Elixir to Splunk. Both Elixir and Splunk have evolving approaches for working with metrics. Sending metrics (which are often stateful, as opposed to log entries, which are stateless) is an additional instrumentation strategy intended to enhance existing real-time insights and observability. This RFC proposes a standardized approach for TID applications to utilize telemetry.

# Motivation

We currently calculate metrics in Splunk by processing logs. This approach is not incorrect, however when the only reason we collect a long entry is to calculate a metric we can save both human and Splunk resources by using a telemetry approach. Because we pay for ingestion, using metrics decreases costs.

Furthermore, metrics are stored as a timeseries meaning they use less storage and require less effort to process than do logs. This makes reporting and alerting speedier. In some cases, it might make possible alerts that previously were too slow to trigger.

# Guide-level explanation

This proposal is primarily concerned with capturing and transmitting both built-in and custom metrics.

First, what is a metric? A metric is an aggregate value.

For example, let's say you want to know how many requests your application is serving per minute. Using logs, you would send a log to Splunk for every single request. You would then query logs and bin entries based on a timestamp. Lastly, you would count each log in the bin. You can do this in Splunk, but it is slow. This is because logs are stored as documents and are not optimized for this kind of data processing.

If you used metrics instead, your application would keep an internal count of the number of requests it was receiving. Every minute it would emit that single number (plus metadata) to Splunk along with the time information. Splunk stores this metric in a timeseries which is optimized for time-based querying.

Phoenix, Ecto, Nebulex, and many other libraries events via [telemetry](https://hexdocs.pm/telemetry/readme.html) providing granular insight into the state of the application. They can be turned into metrics automatically via [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html).

Providing custom metrics is as simple as logging:
```elixir
:telemetry.execute([:web, :request, :done], %{latency: latency}, %{request_path: path, status_code: status})
```

Once your application is emitting metrics, you need to report on them. Multiple reporter libraries exist such as [statsd](https://github.com/beam-telemetry/telemetry_metrics_statsd) or [CloudWatch](https://github.com/bmuller/telemetry_metrics_cloudwatch). You can also write your own as this [example reporter from dotcom](https://github.com/mbta/dotcom/blob/master/lib/cms/telemetry/reporter.ex).

See more: [Logs vs Metrics](https://www.splunk.com/en_us/blog/learn/logs-vs-metrics.html)
See More: [Introduction to Telemetry in Elixir](https://blog.miguelcoba.com/introduction-to-telemetry-in-elixir)

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

There are really two phases to discuss: getting metrics out of our applications and getting those metrics into Splunk.

### Getting metrics out of our applications

For getting metrics out of our Elixir applications, we want to use the [telemetry](https://hexdocs.pm/telemetry/readme.html) library along with [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html) and the [telemetry_poller](https://hexdocs.pm/telemetry_poller/readme.html). The telemetry library standardizes a method for emitting events. The telemetry metrics library turns those events into metrics, and the telemetry poller gathers those metrics and emits them to a reporter at defined intervals.

### Getting metrics into Splunk

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
