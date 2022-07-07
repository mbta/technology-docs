- Feature Name: project-name-rationalization
- Start Date: 2022-05-16
- RFC PR: [mbta/technology-docs#13](https://github.com/mbta/technology-docs/pull/13)
- Asana task: [RFC: rationalizing how we name projects](https://app.asana.com/0/1113179098808463/1202191100361265/f)
- Status: Accepted

# Summary
[summary]: #summary

Resource names in CTD have a consistent pattern.

# Motivation
[motivation]: #motivation

As CTD has grown, multiple teams have independently named their projects and the associated resources. As these names have been generated independently, they are not always consistent about how they use punctuation to separate multiple words in a resource name, leading to confusion.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Within CTD, we use one of the following patterns for naming projects and resources:
- kebab-case: all lowercase, separated by hyphens
- snake_case: all lowercase, separated by underscores
- camelCase: first word is lower case, other words are Title case with no separation
- PascalCase: all words are Title case with no separation

We use the following patterns for resources:
- GitHub Repositories: snake_case
- Elixir modules: snake_case filenames, PascalCase module names (based on the filenames)
	- https://hexdocs.pm/elixir/1.12.3/naming-conventions.html
- Python modules: snake_case
	- https://peps.python.org/pep-0008/
- React components: PascalCase
- TypeScript modules: camelCase
- Shell scripts: kebab-case with no trailing `.sh`
- Domain names: kebab-case
	- https://www.ssl.com/faqs/underscores-not-allowed-in-domain-names/
- S3 buckets: kebab-case
	- https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html
- Terraform names: snake_case
        - https://www.terraform-best-practices.com/naming
- ECS clusters: kebab-case
- ECS applications: kebab-case
- Secrets Manager secret names: kebab-case
- VPC names: kebab-case
- IAM roles: kebab-case
- IAM groups: kebab-case
- IAM users: kebab-case  (but prefer IAM roles whenever possible)
- EC2 security groups: kebab-case
- Cognito user pools: kebab-case
- Cognito app clients: kebab-case
- Glue databases/tables: snake_case
	- https://docs.aws.amazon.com/athena/latest/ug/glue-best-practices.html#schema-names

Abbreviations should be treated as a single word and have their case ignored: gtfs_documentation rather than GTFSDocumentation.

**Exceptions**: 
- OpenTripPlanner (and other forked repositories) should use original repository name and naming conventions
- Elixir module can keep the capitalization of abbreviations (gtfs_documentation.ex -> GTFSDocumentation)
- AWS-created IAM roles are PascalCase (but prefer creating/managing our own via Terraform)
- IAM users for MBTA staff use their AD username
- IAM users for external people use their e-mail address

This RFC does not intend to force existing applications to rename existing names: only to provide standardization as CTD continues to grow and create new projects.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

n/a

# Drawbacks
[drawbacks]: #drawbacks

Standardizing on a pattern likely means choosing something that an individual developer wouldn’t have chosen on their own. However, we feel that the benefits of standardizing to the entire department outweigh individual discomfort in this case.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

These recommendations have been chosen to match existing naming conventions as much as possible. For many of them, any of the other options would be possible. 

# Prior art
[prior-art]: #prior-art

Existing CTD repositories were considered for these recommendations, as were our existing infrastructure. In some cases, language-specific style guides / best practices were also considered.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Are there any other types of resources where we should be standardizing on names?
- Should we be standardizing even further (Credo configurations, Eslint packages, &c)?
- Should documentation repos be allowed to use dashes? (gtfs-documentation and technology-docs, for example)

# Future possibilities
[future-possibilities]: #future-possibilities

Ideally, these naming patterns can be codified into linting rules and enforced during pull requests. 
