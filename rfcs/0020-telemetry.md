- Feature Name: Telemetry

- Start Date: 2024-01-24

- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)

- Asana task: [asana link](https://app.asana.com/)

- Status: Proposed

# Summary

Observability is the ability to measure the internal states of a system by examining its outputs.<sup>[1](https://www.splunk.com/en_us/blog/learn/observability.html)</sup>
It's what allows us to understand how our applications and systems work.
There are three pillars to observability:<sup>[2](https://www.crowdstrike.com/cybersecurity-101/observability/three-pillars-of-observability/)</sup>

  1. **Logs** - are the archival or historical records of system events and errors, which can be plain text, binary, or structured with metadata.
  2. **Metrics** - are numerical measurements of system performance and behavior, such as CPU usage, response time, or error rate.
  3. **Traces** - are the representations of individual requests or transactions that flow through a system, which can help identify bottlenecks, dependencies, and root causes of issues.

TID engineers currently make extensive use of logging.
But, we don't currently have an established mechanism for collecting metrics or traces.
This RFC proposes multiple pathways for collecting metrics in our current observability platform, Splunk.

# Motivation

Logs are handy for debugging and other unstructured research like a security audit.
However, many TID developers have noted that some queries based on logs are incredibly slow, if not impossible, to run.

> It might be tempting to think that logs can solve every use case. As the amount of data grows, however, a logs-only solution will become costly and relatively slow for a small set of regular searches, usually connected to alerts. This is because the process by which logs must be categorized and batched takes much more time and is much more computationally intensive than the metrics process...<sup>[3](https://www.splunk.com/en_us/blog/learn/logs-vs-metrics.html)</sup>

But, as noted above, logs are just one of three tools used to observe systems.
Metrics are optimized for storage and retrieval as numbers in a time series, which makes them cheap to store and fast to retrieve.

By adopting metrics, we can speed up slow queries, decrease the time necessary to generate dashboards and reports, and make previously impossible alerts possible.

Furthermore, we can decrease costs by reducing the data we send to Splunk.

# Guide-level explanation

This proposal primarily concerns capturing and transmitting metrics.

What is a metric, and how is it different from a log? Let's define a log as a structured **event** detailing information about a program's execution. Conversely, a metric is a single **number** valuable for informing us about a program's execution. Logs and metrics are helpful in different ways.

For example, you want to know how many requests your application served per minute over the last hour. Using logs, you would send a log to Splunk for every request. Dotcom regularly receives 150 requests per second or 9,000 requests per minute. So, if you wanted to look at the span of an hour, you would query 540,000 logs and bin each log by its timestamp. Lastly, you would count each log in each bin. You can do this in Splunk, but it is slow. It gets slower and slower as you increase the time horizon or amount of data because logs are stored as documents optimized to be searched rather than calculated.

If you used metrics instead, your application would keep an internal count of the requests received. Every minute (tuneable per application), it would emit that number and a timestamp to Splunk before resetting itself. So, in a minute, it would emit the number 150 once. That is one request instead of the 9,000 above. Reviewing an hour of data would entail nothing more than looking at 60 numbers. Metrics are a much better approach to answering the question.

Elixir libraries such as Phoenix, Ecto, Nebulex, and many others have adopted the [telemetry](https://hexdocs.pm/telemetry/readme.html) library as a standardized way to give users observability into their performance. It emits events that can be turned into metrics automatically via [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html).

Providing custom metrics is as simple as logging:
```elixir
:telemetry.execute([:web, :request, :done], %{latency: latency}, %{request_path: path, status_code: status})
```

Once your application is emitting metrics, you need to report on them. Multiple reporter libraries exist, e.g., [statsd](https://github.com/beam-telemetry/telemetry_metrics_statsd) or [CloudWatch](https://github.com/bmuller/telemetry_metrics_cloudwatch). You can also write your own, such as this [custom reporter from dotcom](https://github.com/mbta/dotcom/blob/master/lib/cms/telemetry/reporter.ex).

See More: [Introduction to Telemetry in Elixir](https://blog.miguelcoba.com/introduction-to-telemetry-in-elixir).

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

As was mentioned previously, there are two phases to discuss: getting metrics out of our applications and getting those metrics into Splunk.

## Getting metrics out of our applications

We want to use the [telemetry](https://hexdocs.pm/telemetry/readme.html) library, [Telemetry.Metrics](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html), and [telemetry_poller](https://hexdocs.pm/telemetry_poller/readme.html). The telemetry library standardizes a method for emitting events. The telemetry metrics library turns those events into metrics, and the telemetry poller gathers those metrics and emits them to a reporter at whatever intervals you define. You can see an [example implementation from dotcom](https://github.com/mbta/dotcom/blob/master/lib/cms/telemetry.ex).

## Getting metrics into Splunk

We recommend two options for getting metrics into Splunk, and teams should feel free to pursue the path that best fits their application and expertise.

**Note**

There is no reason you cannot combine the two methods.
You could use the HEC for your Elixir applications and the UF for non-Elixir applications.

### Splunk Universal Forwarder

The Splunk Universal Forwarder (UF) is an agent that runs alongside an application, takes in metrics in one of many formats, and forwards those metrics to Splunk.
The UF can be used with any application, regardless of language, which makes it the most flexible way to get metrics into Splunk.

It runs in a [sidecar](https://www.oreilly.com/library/view/designing-distributed-systems/9781491983638/ch02.html) to any container that emits metrics.

To use it, create a metrics index in Splunk and add the sidecar to the `container_definition_json` of your `aws-ecs-container-definition` module:
```terraform
container_definition_json = jsonencode([
    module.your-container.json_map_object,
    module.sidecar-container.json_map_object
])
```

You use the StatsD telemetry reporter highlighted above to emit your metrics in the StatsD format, and the UF will automatically forward them.

#### When you should choose this method

- You maintain applications that aren't written in Elixir or want all of your applications to emit metrics in a standard format.

Every language has a library that emits metrics in the StatsD format; they are straightforward to set up.
You get a lot of flexibility with very little work by running the UF as a sidecar and using an intermediary standard like StatsD.

#### When you shouldn't choose this method

- You only need metrics from Elixir applications.
- You are hesitant to change your infrastructure.
- You are concerned about your ECS resource usage.

The UF sidecar uses CPU and memory resources.
Though the footprint is small, you might be worried that increasing your compute usage might affect the rest of your cluster.
Resource usage is more pertinent if you run resource-lite instances, as the UF would take up a more significant percentage of CPU and memory.
Note that you can always increase CPU and memory levels for your cluster.

### Splunk HTTP Event Collector

The Splunk HTTP Event Collector (HEC) allows us to submit metrics via HTTP.
We maintain a library [TelemetryMetricsSplunk](https://github.com/anthonyshull/telemetry_metrics_splunk) that forwards telemetry metrics to Splunk via the HEC.

To use it, create a metrics index in Splunk and add its HEC token to the library configuration.

```elixir
config :app, TelemetryMetricsSplunk,
  index: "your-index"
  token: System.get_env("SPLUNK_TOKEN")
```

#### When you should choose this method

- You only need metrics from Elixir applications.

#### When you shouldn't choose this method

- You maintain non-Elixir applications and do not want to write a method of getting metrics into Splunk.

There aren't any libraries in other languages that transmit metrics to Splunk via the HEC.

# Drawbacks

[drawbacks]: #drawbacks

You can argue that we do not need metrics and that adding a means to collect them increases the complexity of our technology stack.
However, we hope the limitations of a logs-only approach to observability have been made apparent by this RFC.
Furthermore, we hope that the promised benefits of metrics cause excitement.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

### AWS Cloudwatch

We considered using AWS CloudWatch as an intermediary for metrics.

#### Positives

Like the StatsD reporter and our HEC reporter, an AWS CloudWatch reporter already exists.
It is possible to send metrics to CloudWatch and then ingest those into Splunk.

To use it, teams must set up CloudWatch, set the correct permissions, and set the configuration in Splunk.

See more: [Configure CloudWatch inputs for the Splunk Add-on for AWS](https://docs.splunk.com/Documentation/AddOns/released/AWS/CloudWatch).

#### Negatives

AWS charges for every metric stored in CloudWatch.
It also charges for every request for a metric.
So, by using CloudWatch, we would be paying to store metrics in CloudWatch, paying to transfer them from CloudWatch to Splunk, and then paying Splunk to ingest them.

As a paper napkin exercise, consider dotcom's monitor application.
It emits 15 metrics every minute--21,600 every day.
AWS charges one penny for every 1,000 metrics requested.
Thus, it would cost $2.16 a day or $788.40 per year to transfer these 15 metrics from CloudWatch to Splunk.

See more: [Example: Cost scenarios using polling APIs](https://docs.splunk.com/observability/en/infrastructure/monitor/aws-infra-costs.html#aws-costs-amazon).

# Future possibilities

[future-possibilities]: #future-possibilities

We have only addressed two of the three pillars of observability and have not considered [tracing](https://www.atatus.com/blog/logging-traces-metrics-observability/#tracing).
Tracing allows us to observe the complete journey of a request or workflow as it moves from one part of the system to another.
Our current Splunk instance does not support tracing as we need an add-on [Splunk Observability](https://www.splunk.com/en_us/products/observability.html).

There are also myriad alternatives, such as [Honeycomb](https://www.honeycomb.io/) and [SigNoz](https://signoz.io/).

These solutions and Splunk Observability support [OpenTelemetry](https://opentelemetry.io/), an industry standard for observability across languages.
The [OpenTelemetryTelemetry](https://github.com/open-telemetry/opentelemetry-erlang-contrib/tree/main/utilities/opentelemetry_telemetry) library can convert Telemetry (the Elixir library) events to OpenTelemetry spans.