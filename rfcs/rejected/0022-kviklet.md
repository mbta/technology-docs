# Kviklet 

- Feature Name: `kviklet`
- Start Date: 2024-03-29
- RFC PR: [mbta/technology-docs/#0022](https://github.com/mbta/technology-docs/pull/22)
- Asana task: [Kviklet (run reviewed DB queries on prod)](https://app.asana.com/0/1200506724882024/1206873943983582/f)
- Status: Rejected

## Summary

Set up [Kviklet](https://kviklet.dev/), a self-hosted tool to allow teams access to the production database to run one-off queries, after review by other team members, similar to code review.

The status-quo and alternative is that teams can only run database queries by deploying code to prod.

## Motivation

I need the ability to run arbitrary read queries against the Glides prod database, in order to:
- Debug issues in prod. Examples:
  - [Is a field that we think is unique actually unique in practice?](https://app.asana.com/0/1200273269966439/1205996566957450)
  - Gather quantitative data about realtime tracking problems, instead of just ad hoc reports when somebody notices them.
  - Sign in isn't working for only one operator. Do we have their info in the database?
- Run Analytics. Examples:
  - [What are people putting into free-text comments, and does it suggest any features we should build for them?](https://app.asana.com/0/616151179860796/1206468619394322)
  - How long before a trip are inspectors entering departure times? (This affects prediction quality, and has been a major effort of Glides that we've been iterating on for a while, but it's harder when it's hard to export data.)

This is not just a problem for Glides. Any app with a database is likely to run into similar problems.

This has been a problem for us for a long time, which we haven't found a good solution for yet.

Kviklet wrote a blog post about why it's good for developers to have (careful) prod access: [Should Developers have production access?](https://kviklet.dev/blog/should-engineers-have-production-access)

## Status Quo / Alternative

The prod database can only be accessed by the application, so in order to run a query, the Glides team must deploy a code change to the server. This is pretty high overhead that must be paid even for a small one-off query that takes 30 seconds to plan and run locally.

Currently, the Glides app has an admin UI with a way to run a few hand-written queries. Adding or changing these is not easy, because it requires updaing UI and API endpoints in addition to writing the query itself.
 
When [I asked other teams](https://app.asana.com/0/1200506724882024/1206647196651141/f), the recommended approach was to build a system like T-Alerts, where they still have to deploy code with a hardcoded approved query, but have a better system set up to reduce the effort and overhead in writing new queries. Glides was planning on doing this approach until we learned of Kviklet.

Other options that were considered and rejected in the asana task linked above:
- An endpoint in the app to run arbitrary queries without a deploy: Vulnerable to SQL injections / huge security hole.
- Direct prod access (with a read-only postgres user): Can't control PII access.
- Read replica: Complexity and cost to maintaining a second DB.
- DMS database dumps: Have had trouble scrubbing PII from them.
- Lambda to read from the DB: Similar effort to doing it through approved app queries, it just trades overhead of deploying the app to overhead of managing the lambda.

## What is Kviklet

[Kviklet](https://kviklet.dev/) is a tool for reviewing and running queries against the prod datasbase. It's free, open source (MIT license), and self hosted. It requires review before running a query, similar to a pull request, and leaves an audit trail.

It can be configured to require different amounts of review when touching different databases (for example, requiring extra review before accessing databases with PII).

It supports Keycloak SSO (though we'd need to do role management within Kviklet).

We would only give Kviklet read access to our databases, not write or admin access, to reduce risk. Any write queries would continue to have to go through an application deploy, like today.

## Benefits of Kviklet compared to alternative

- Will be much faster and easier to run queries.
- Is full of features that are meant for database access, such as audit logs, previewing an query plan, saving queries and results, granting only temporary access, etc.
- Would scale well to be usable by all teams, instead of each team needing to build its own custom UI for running queries.
- Could be usable by PMs who want to write their own SQL queries (with review by engineers).

## Costs/Risks of Kviklet

- The effort of setting it up, maintaining the infrastructure for it, and using the app (managing permissions, etc).
  - The infrastructure team is already overburdened, and this would add to their workload in the short term.
- It would be a centralized place with access to multiple of our databases, which is a security risk.
- It doesn't support IAM auth to the database, so would require managing a Postgres user and password, which is more work and less secure.
- Less customizable than writing custom code and UI to run queries.
- It's a relatively new and unproven application, run by only 2 people. (It was going to be a business but [they decided to open source it instead](https://kviklet.dev/blog/kviklet-is-now-mit-licensed).)

## Details about setting up Kviklet

[Setup docs](https://github.com/kviklet/kviklet?tab=readme-ov-file#setup)

Kviklet runs as a docker container that we deploy on our own AWS infrastructure. We would also create a new PostgresDB for it. We would need to configure it to connect to keycloak, and then set up roles from within the app.

Unfortunately, Kviklet doesn't support IAM auth. To connect to the application databases, in order to avoid giving Kviklet write access, each database would need a new postgres user with only read permissions for Kviklet to use.

The initial set up can be mostly done by the Glides team opening Terraform PRs. However, the infrastructure team will be needed to:
- Review and advise on the terraform configuration.
- Execute `terraform apply`.
- Create a new Postgres user.

Glides would be the first application to use it. Assuming it goes well, other apps could then also connect their apps to Kviklet, as they have a need for it. 

Once other teams want to get set up, ongoing mainteance tasks would mostly become the infrastructure team's responsibility, and would include:
- Deploying new versions as they come out (it's under active development).
- User management, as teams change.
- Connecting new app databases, if more teams decide to begin using Kviklet for DB access.

## Timeline

Setting up Kviklet is not urgent. Although the Glides team is feeling pain today from not having a good solution for running DB queries, there are workarounds. This proposal is mostly motivated by long-term process improvement rather than an urgent problem. It can go through the infrastructure team's normal planning process.

I'm not sure exactly how the infrastructure roadmap process works, but I would hope for this to be done in the next quarter that the infrastructure team has not already filled out their quarterly plan for.
