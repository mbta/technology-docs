- Feature Name: Telemetry

- Start Date: 2024-01-24

- RFC PR: [mbta/technology-docs#0000](https://github.com/mbta/technology-docs/pull/0000)

- Asana task: [asana link](https://app.asana.com/)

- Status: Proposed

  

# Summary

TID engineers currently make extensive use of logging for a wide range of use cases, primarily from Elixir to Splunk. Both Elixir and Splunk have evolving approaches for working with metrics. Sending metrics (which are often stateful, as opposed to log entries, which are stateless) is an additional instrumentation strategy intended to enhance existing real-time insights and observability. This RFC proposes a standardized approach for TID applications to utilize telemetry.

# Motivation

We currently calculate metrics in Splunk by processing logs. This approach is not incorrect, however when the only reason we collect a long entry is to calculate a metric we can save both human and Splunk resources by using a telemetry approach. In theory, this approach uses less bandwidth and less storage and requires less effort to process in Splunk compared to log entries.

# Guide-level explanation

This proposal is primarily concerned with capturing and transmitting both built-in and custom metrics. First, what is a metric? A metric is an aggregate value. For example, in a log entry, you might send a single HTTP request to Splunk. Later, you could count those requests and generate metric about requests per minute. It is also possible to use a telemetry approach where the count of requests gets incremented prior to getting transmitted to Splunk such that the metric doesn't require post-processing to become a count.

Phoenix, Ecto, and potentially other libraries have built-in metrics that can be transmitted as telemetry, providing more insight into the operations of our systems at a more granular level.

When there is a need for a custom metric, an engineer should learn about the available types of metrics and consider if the telemetry approach would be a good fit, as well as understand how to view and visualize metrics in Splunk.

Depending on the systems an engineer may have worked on previously, it is not uncommon to have used telemetry in the past, and some folks may be quite familiar with this approach.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

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
