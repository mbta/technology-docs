- Feature Name: consistent-environment-naming
- Start Date: 2022-09-14
- RFC PR: [mbta/technology-docs#0015](https://github.com/mbta/technology-docs/pull/15)
- Asana task: [Dev environment naming / usage conventions](https://app.asana.com/0/1200506724882024/1202675232761300/f)
- Status: Proposed

# Summary
[summary]: #summary

Non-production environments of CTD applications have clear, descriptive, and consistent naming.

# Motivation
[motivation]: #motivation

Non-production environments for our applications are generally named with some variation on `dev`. This doesn't capture the distinctions between different non-production uses of application environments.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Environments are named in `kebab-case`, following the convention `[name of application]-[modifier]`. Available modifiers are as follows, with a brief description of their use case:

- `prod`: the production environment.
- `staging`: used for final integration testing and monitoring for errors before deploying changes to production. Often, merging a PR into the `main` / `master` branch will be set up to automatically initiate a deploy to the application's `staging` environment. There should be one stage environment per application.
- `test`: used as the target environment for some sort of recurring automated testing, such as end-to-end web application testing or load testing. Accepts an optional modifier specifying what testing tool the environment is for (for instance, `test-cypress` or `test-locust`). This is presently a hypothetical use case, albeit one that has been discussed from time to time.
- `dev-[color]`: for experimenting with and evaluating features under active development, generally in their own branch. Potential uses include: allowing a developer to experiment with code that relates to functionality that's hard to replicate locally, providing a space for other team members to evaluate and sign off on a feature before it is merged, and collecting application-specific metrics on a particular change before deciding whether to merge it. `[color]` can be the color of any MBTA rapid transit line (`green`, `red`, `orange`, `blue`, or `silver`). Alternatively, if a development environment is created temporarily for testing a specific feature, a short, descriptive name for that feature may be substituted for the color.

Upon acceptance of this RFC, the New Application Checklist will be updated to dictate use of this new naming scheme. Converting the naming of existing environments to reflect this scheme will be gently encouraged but not required on any particular timeline.

## Data pipeline

Given that CTD maintains many interrelated applications that exchange data, the output of other applications is often very important to determining how a given application will behave. As such, it is important for testing purposes that engineers and other team members understand these relationships, and that the data sources for a given environment properly facilitate its usage. Given these considerations:

- `prod` environments should only ever get their data from other `prod` environments.
- `staging` environments should only ever get their data from other `staging` environments or `prod` environments, depending on which is most appropriate for monitoring application stability.
- `test` environments should generally default to getting their data from `staging` environments. In cases where there is some end-to-end test process across multiple applications, it may make sense to have one `test` environment get its data from another application's `test` environment.
- `dev-[color]` environments should generally default to getting their data from `staging` environments. When multiple teams are working on interrelated features (for instance, TRC is introducing a data source and dotcom is making updates to display that data), it may make sense to point one `dev-[color]` environment to another application's `dev-[color]` environment. The colors of the dev environments in the different applications need not necessarily match up, although if possible that may be desirable to make the configuration easier to remember.

As noted, there are cases where there is a general default guidance for which data sources to use, but there is room for variation. When deviating from the default, make sure that this is documented in the [Miro board](https://miro.com/app/board/uXjVOcLVgUY=/) that shows how environments talk to each other.

Some data sources aren't CTD-maintained applications, but rather outside systems we interface with. Examples include OCS messages for subway tracking, or Alerts UI for creating and managing alerts. In these cases, the names of the environments won't necessarily match ours, and there may not be as many development environments as we have, or even any non-production environment at all. In these situations, engineers should make a judgement call as to which one is the best match in each case.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Having explained the naming scheme above, this section will go into more detail about the actual transition.

The main implementation step that will be needed is that applications with existing `dev` environments will need to rename them to `staging`. The easiest, and recommended, approach is to simply create a new environment in Terraform under the new name and delete the old one. If there is any application state in a database that needs to be carried over, this can be handled by using this [process to snapshot and restore](https://github.com/mbta/wiki/blob/master/devops/rds-database-maintenance.md#snapshot--restore) the database contents. Even for many applications with a database, though, the database contents on non-production environments are expendable and this step can be skipped.

In addition to the names of AWS resources and hostnames, care should be taken to make sure that any resources that a given application environment publishes (for example, an enhanced GTFS-rt feed in JSON format uploaded to S3) and renamed accordingly.

Applications that currently have a `dev` but no `dev-[color]` environment are generally encouraged to also add at least one `dev-[color]` environment if they are still under active development.

# Drawbacks
[drawbacks]: #drawbacks

Environment names will need to be updated in a lot of places (Terraform, GitHub Actions, any saved links, and so on). Furthermore, given the fact that so many of our applications are interdependent, changing the name of an environment in one application may require adjusting something in the configuration of an environment for a different application. These drawbacks may outweight the benefit of increased clarity in environment naming.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Regarding the actual choice of names, `staging` is probably the one most open to other suggestions. Ultimately, there are a few reasons I went with `staging` as opposed to some other name:

- It emphasizes this environment's role in the merge and deploy process.
- I considered `test` as well, but since this can imply some more specific kind of testing is going on rather than simply running and monitoring the application, I opted to instead use that term for future hypothetical environments for supporting automated testing tools.
- Another possibility considered was `qa`. However, most teams perform validation and QA of features before merging. In addition, we don't have a strong reliance on manual QA or a separate QA discipline to begin with, so it's not a widely-used term in CTD.

As for the decisions around data pipeline, my goal was to at least provide some defaults that keep things simple in the default case while providing flexibility where appropriate. Currently, when creating a new dev environment, most decisions about which version of the API or other data sources to point to are somewhat arbitrary. Having a clean distinction between `staging` and development environments that are intended for more interactive, but also ephemeral, use cases makes it clearer that those should be the default data sources for most non-prod environments.

# Prior art
[prior-art]: #prior-art

The main piece of prior art that this RFC draws from is the use of "staging" as terminology to describe the use case for most of our current "dev" environments. This appears to be fairly common in the industry; it was in use at my previous company, and other CTD engineers have understood what I mean by it when discussing this issue.

CTD has already been using `dev-green` as an environment name, as well as `dev-blue` in a handful of cases. This naming dates from when CTD was experimenting with [blue-green deployments](https://martinfowler.com/bliki/BlueGreenDeployment.html), a practice that was not ultimately adopted. Since these are already colors of MBTA rapid transit lines, it can be reworked to provide the basis for an expanded naming scheme that's also thematic. Many teams will probably only need one or two `dev` environments, but larger teams that work on a lot of features simultaneously may benefit from more, and this proposal provides convenient names for them.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Will there be any impact on non-production environments of the various vendors that interact with our data pipeline, like Swiftly?

# Future possibilities
[future-possibilities]: #future-possibilities

This naming convention should hopefully provide a stable foundation going forward, but if the need for new, different kinds of environments arises we could certainly add naming conventions for them.
