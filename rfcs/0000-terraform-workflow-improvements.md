- Feature Name: Terraform Workflow Improvements for 2022
- Start Date: 2022-10-24
- RFC PR: [mbta/technology-docs#16](https://github.com/mbta/technology-docs/pull/16)
- Asana task: [Terraform Workflow Improvements RFC](https://app.asana.com/0/1113179098808463/1202075979488110/f)
- Status: Proposed

# Summary
[summary]: #summary

<!--
One paragraph explanation of the feature.
-->

Over the last two years, Terraform has had a significant impact on how CTD manages the infrastructure for our applications. Terraform gives engineering teams a clearer view of each application's underlying AWS resources, and enables engineers to handle management of infrastructure configuration more directly. However, for most of 2022, we've been at an inflection point; in many ways, the needs of the team have scaled beyond our current processes for managing Terraform configuration. There are a number of improvements we need to make to scale our workflow, including:

- Giving engineering teams smaller, more focused areas of Terraform configuration to work in
- Improving our Terraform testing workflow
- Standardizing the more commonly used infrastructure components
- Potentially versioning parts of our Terraform configuration
- Automating more of the work of interacting with Terraform runs
- Resolving some of the issues around user permissions while maintaining the security of our infrastructure

# Motivation
[motivation]: #motivation

<!--
Why are we doing this? What use cases does it support? What is the expected outcome?
-->

The ECS migration project uncovered numerous pain points for engineers working with Terraform. In interviews conducted after that project was completed, teams shared a lot of useful feedback, which has been condensed into the following issues of focus:

**Scalability**

- Only one engineer at a time can run `terraform plan` in the `dev` root module
- Plan output often shows information not relevant to the engineer/team

**Testing**

- The way we're testing changes&mdash;using [Terraform workspaces][terraform-workspaces]&mdash;isn't very effective
- When working on application modules, running `plan`/`validate` requires switching between root module + child modules

**Permissions**

- The `dev` root module is hard to develop against for people who lack all the required AWS permissions
- Testing changes requires applying/reverting, which isn't always a viable workflow depending on user permissions and the resources involved

Additionally, suggestions for possible improvements were offered by engineers, including:

- Setting up per-team environments or modules
- Moving towards "an architecture where engineers can make changes without affecting completely unrelated changes”
- Running `terraform plan` automatically in some way
- Abstracting more reusable components
- Running more targeted plans

The improvements proposed in this RFC aim to directly address these issues and ideas.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

<!--
Explain the proposal as if it was already implemented and you were teaching it to a new developer
that just joined the team. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how programmers should *think* about the feature, and how it should impact the way they
  work on this project. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to senior developers and to junior
  developers.
-->

To address the scaling issues and pain points called out in the [summary][summary] and [motivation][motivation] sections, the following improvements are recommended:

1. Splitting root modules into smaller, more focused parts
2. Implementing a testing workflow for Terraform modules
3. Standardizing on modules for commonly used infrastructure components
4. Defining a process for module versioning and publishing, where applicable
5. Automating Terraform runs through the use of a third-party tool such as Scalr
6. Ensuring appropriate permissions that allow teams to self-manage infrastructure in a secure way

The following subsections detail each improvement broadly. For specific implementation details, see the [reference-level explanation][reference-level-explanation] section below.

## 1. Smaller, More Focused Modules
[guide-module-splitting]: #1-smaller-more-focused-modules

_Too Many Cooks &rarr; Multiple Kitchens_

Since many applications have multiple development and staging environments, there are more resources overall to manage in the development context. Managing all these resources&mdash;over 1,600 at the time of this writing&mdash;in a single module makes Terraform run slower, since it has to refresh the state of all resources every time it's run. For that matter, non-production resources see more frequent changes than production to facilitate feature development, which means more Terraform runs, and each run requires locking the state for an extended period of time.

A common approach with Terraform is to split configuration into multiple modules instead of managing everything in one central module. This approach was recommended by every [Terraform automation][guide-automation] vendor we talked to, including HashiCorp. CTD already has multiple root modules for different types of configuration, but as we scale up, we need more.

A sensible and straightforward solution is to split development resources by team. Doing so has many benefits:

- Mitigates state lock contention by giving each team their own state
- Reduces overall number of resources per module for faster plan time
- Each team (or the [automated tool][guide-automation] acting on their behalf) only needs AWS permissions to manage its own resources

For more details on how this will be accomplished, see the corresponding [reference-level explanation][reference-module-splitting].

## 2. Testing Workflow
[guide-module-testing]: #2-testing-workflow

_Test in isolation_

To date, the testing of Terraform changes at CTD was done using [Terraform workspaces][terraform-workspaces]. The use of workspaces is problematic in that they necessitate a confusing process of duplicating Terraform state, yet they still ultimately manipulate resources running in our main AWS account, which can cause drift in the default Terraform state while the testing is being performed.

When testing software, it is a best practice to use an isolated environment and provide a specific set of configuration designed specifically for carrying out the tests. Testing infrastructure should be no different &ndash; We should be able to create a new, isolated configuration specific to testing each module, and use Terraform's `apply` and `destroy` commands to bring it up and tear it down, respectively.

Thankfully, the majority of the infrastructure maintained by engineering teams is defined in independent _modules_ (also sometimes referred to as _child modules_ to differentiate them from the _root modules_ that directly manage infrastructure state). Each application's module defines a set of infrastructure components, either by defining Terraform resources directly, or calling other modules that define infrastructure components.

The use of modules is a Terraform best practice; modules allow engineers to define all of an application's infrastructure as a self-contained set of configuration, which can then be used to create multiple nearly identical environments. Since modules are intended to be largely independent and reusable, it's relatively easy to create and destroy a module's resources separately from the context where the module will ultimately be deployed.

In short: **To test our infrastructure configuration in Terraform, we should focus on testing modules.**

A convention for module testing [recommended by members of the Terraform community](https://discuss.hashicorp.com/t/best-practices-of-terraform-staging-testing/6762/2) is to create a `test/` subdirectory within each module directory to store its test configuration. The test module defines any dependent resources required by the module, and then calls the module itself. It can even call other application modules if needed.

To truly enforce the "isolation" requirement, module testing will be done in a fully separate AWS account where we can carry out the creation and destruction of test resources independently from our existing infrastructure. This will help us avoid the possibility that testing resources interfere with production systems in any way.

For details on the proposed testing process, see the corresponding [reference-level explanation][reference-module-testing].

## 3. Standardize Components
[guide-standardization]: #3-standardize-components

_Don't repeat yourself_

CTD's applications largely share a common architecture, which in turn means that most applications use the same types of AWS resources. Specifically: Most of our applications run as ECS services, many include a load balancer, and some also store data in an RDS database. These components largely have common configuration, save for a few attributes that need to be customized for each application.

Recently, effort has been put toward defining Terraform modules that encapsulate as much of the configuration as possible for these infrastructure components. Those modules are:

- `ctd-ecs-app` for ECS service and load balancer configuration
- `ctd-rds-db` for RDS database configuration

These _infrastructure component modules_ have seen some adoption, but are not yet widely used. To maximize their effectiveness, we will:

- Set up a testing workflow for each module using the process described in the [Testing Workflow][guide-module-testing] section
- Move them to separate repos and version them, using the process described in the [Version and Publish Modules section][guide-module-versioning]

We will also push to migrate all applications to use these modules to minimize redundant and nonstandard configuration.

Relatedly, the `ctd-ecs-app` module introduced the concept of a `context` var that bundles all baseline configuration need by a module, including VPC and subnet information, and shared resources such as TLS certificates and Cognito domains. This convention has the potential to significantly reduce redundant configuration in app modules, and should also be adopted widely.

See the corresponding [reference-level explanation][reference-standardization] for related recommendations.

## 4. Version and Publish Modules
[guide-module-versioning]: #4-version-and-publish-modules

_Modules as immutable artifacts_

Nearly all of CTD's modules are currently stored in the devops repo and called using a relative path to the module's directory. Terraform supports a number of alternate methods for sourcing modules, including:

- Git repositories (including specific refs)
- Public module registries
- Private module registries

The main advantage of using a repo or module registry instead of a path is that the module can then be treated as a versioned artifact, and deployed separately from the configuration that references it. This makes it easier to add new functionality to a module without having to implement a feature gating mechanism (such as optional module variables, which is the typical approach currently). Versioning is potentially useful for both infrastructure component modules and application modules, as a way to make changes to a module without immediately impacting all the contexts in which it's used. Each call to the module could then be upgraded separately as needed.

However, supporting module publishing is not without its caveats; in order to publish a module, it must:

- Have a process for publishing a git release/tag for each version
- Follow a strict repo or directory naming convention: `terraform-<PROVIDER>-<NAME>`

On top of that, in order to publish a module to Terraform's public registry, it must adhere to [a strict set of requirements][terraform-module-publishing], not least of which being that it must be stored in a discrete public repo. Private registries are less restrictive on these points, but are only available through paid platforms such as Scalr.

Consider also that publishing modules is not strictly required in order to pin a module call to a specific version, since it's possible to use a git hash as a version string when calling a module from a repo. Still, there is merit to implementing a versioning and publishing workflow for infrastructure component modules, since they are used more broadly across different applications, and version strings are more comprehensible than git hashes.

In summary: We should prioritize setting up a versioning and publishing workflow for infrastructure component modules in either a public or private registry, but given the additional overhead, it should be considered an optional improvement for application modules.

For details on the module versioning and publishing workflow, see the corresponding [reference-level explanation][reference-module-versioning].

## 5. Terraform Automation
[guide-automation]: #5-terraform-automation

_Automate as much as possible_

Even when broken up by team, root modules would be easier to use if there were a central system to handle runs, manage state locks, and hold the permissions required to make changes. Similarly, a workflow for testing modules would be significantly more effective if it could be run automatically.

The CTD Infrastructure team has investigated and evaluated different products for automating Terraform. Based on our investigation, we think [Scalr](https://www.scalr.com/) is the optimal solution for CTD's needs, and we are working with the CTD Finance team to procure it as soon as possible. (A [list of other products that were evaluated][alternatives-automation] is included in the rationale and alternatives section.)

Scalr stands to improve our use of Terraform by:

- Centralizing plan/apply runs to mitigate locking issues
- Working as a remote operation backend to reduce reliance on each engineer's workstation for runs, and their network connection for AWS API operations
- Owning the delegated AWS permissions required to run `terraform plan` and `terraform apply`
- Providing an additional layer of permissions on top of AWS to manage access for planning and applying different root modules
- Employing [Open Policy Agent][open-policy-agent] (OPA) checks to require approval before applying certain types of resources (e.g. IAM role and policy changes)

Scalr can be configured to work in tandem with GitHub, such that pull requests automatically run `terraform plan` on creation, and `terraform apply` on merge. In many cases engineers can still run Terraform commands manually, using Scalr as a remote backend to handle execution and AWS permissions. Existing Terraform checks that are in place, such as configuration validation and formatting, will continue to be run by GitHub Actions.

Note that CTD's ability to procure Scalr is not yet confirmed for FY23, so there are some aspects of this proposal that could shift depending on how soon we're able to make use of Scalr as an automation tool. This possibility is [covered in detail][alternatives-automation] in the rationale and alternatives section.

For details on how Scalr will be configured, and the proposed CI/CD workflows, see the corresponding [reference-level explanation][reference-automation]. There are also some caveats outlined in the [drawbacks section][drawbacks-scalr].

## 6. User Permissions
[guide-permissions]: #6-user-permissions

_Strike a better balance_

When managing access to manipulate AWS resources, it's important to balance usability with security. On the usability side, teams should generally be able to review each other's work and apply it themselves, and the infrastructure team shouldn't always be a blocker to carrying out this process.

But on the security side, there is a very large elephant in the room when it comes to AWS permissions, and that is IAM. **We cannot allow engineers to directly manage IAM permissions in an unrestricted manner**, lest they be able to unknowingly or maliciously escalate privileges and undermine the security of our AWS infrastructure.

With that in mind, our goal for all of the proposed improvements will be to maximize the effectiveness of teams working with Terraform while still maintaining a secure environment. This means that any Terraform changes that include IAM permissions will still require infrastructure approval. It will also likely mean excluding certain IAM permissions from Scalr and/or engineering user groups, possibly including those that allow deletion of certain types of resources.

For more details on how we'll accomplish this, see the corresponding [reference-level explanation][reference-permissions].

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.
-->

## 1. Splitting Root Modules By Team
[reference-module-splitting]: #1-splitting-root-modules-by-team

To optimize the management of development and staging resources, the `dev` root module will be split into multiple root modules that are organized by team. Resources that are shared by some or all applications/teams will be moved to the `global` module, which will be renamed to `aws-ctd-main-base`. To facilitate references to these common resources, the `context` var will also move to the `aws-ctd-main-base` module, and will be defined as an [output value](https://developer.hashicorp.com/terraform/language/values/outputs) that can be accessed by team-based root modules.

Once a new root module is set up for each engineering team, we'll need to:

- Migrate each team's development and staging resources out of the `dev` module and into their new team-based module
- Update references to dependent resources that remain in other modules to use data sources or [remote state](https://developer.hashicorp.com/terraform/language/state/remote-state-data)

Note that production resources will remain in a single `prod` root module in the short term, but will likely eventually also be moved to team-based modules a later date. This will allow us to vet the changes proposed in this document, while also making sure we're adhering to the stricter security requirements for production that are discussed in the [Scalr workflow for production][scalr-workflow-production] section.

### Root Module Naming Convention
[root-module-naming-convention]: #root-module-naming-convention

We'll also need to implement a new naming convention for root modules to keep them organized.

Terraform's official [naming convention for published modules][terraform-module-publishing] is `terraform-<PROVIDER>-<NAME>`, where `<NAME>` can contain additional hyphens. Since our own root modules won't be published to Terraform's public registry, we can omit the `terraform-` prefix and use `<PROVIDER>-<NAME>` as a base convention, where `<NAME>` can contain any of the following:

- AWS account nickname (e.g. `mbta-ctd-test`, `mbta-ctd-main`, or preferably a more succinct version that eliminates the `mbta-` prefix)
- Team name (prefixed with `team-`)
- Environment context (e.g. `dev`, `staging`, `prod`)
- Component descriptor of the type/category of resources the module is intended to contain, if it's one of multiple modules for the particular provider/account/context (e.g. `base`, `bootstrap`, `restricted`)
- Nothing (in cases where one module encompasses all of a provider's configuration)

A side-note regarding the AWS account nicknames: Our main AWS account is currently named "mbtacomsupport", but we are hoping to [rename it](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-alias.html) soon. We toyed with the name "mbta-ctd-prod", but since the account also contains development/staging resources&mdash;which can't reasonably be provisioned into their own separate "development/staging" AWS account&mdash;we've opted to recommend the name "mbta-ctd-main". That name is open to debate, however.

#### Proposed Module Names

With all the above in mind, the following are the proposed names for all existing and future root modules. (Note that the naming convention for child modules is different, and is covered in the [module versioning][guide-module-versioning] section.)

Root modules associated with the "CTD Production" AWS account that should be renamed:

- `bootstrap` → `aws-ctd-main-bootstrap`
- `global` → `aws-ctd-main-base`
- `dev` → multiple modules:
  - move application development/staging resources to team-based modules:
    - `aws-ctd-main-team-dataplatform`
    - `aws-ctd-main-team-glides`
    - `aws-ctd-main-team-lamp`
    - `aws-ctd-main-team-pass-programs`
    - `aws-ctd-main-team-screens`
    - `aws-ctd-main-team-trc`
    - `aws-ctd-main-team-website`
  - move base dev resources to `aws-ctd-main-base`
- `onprem_dev` → multiple modules:
  - move SSM resources to `aws-ctd-main-base`
  - move applications to team-based modules
- `onprem_prod` → multiple modules:
  - move SSM resources to `aws-ctd-main-base`
  - move applications to `aws-ctd-main-prod` for now
- `prod` → `aws-ctd-main-prod` (to remain as a single module for now)
- `restricted` → `aws-ctd-main-restricted`

Root modules to remain as-is:

- `scalr`

New root modules for "mbta-ctd-test" AWS account:

- `aws-ctd-test-base`
- `aws-ctd-test-bootstrap`
- `aws-ctd-test-restricted`

Possible future root modules:

- `github`
- `sentry`
- `splunk`

## 2. Module Testing, Detailed
[reference-module-testing]: #2-module-testing-detailed

As noted in the [guide-level explanation][guide-module-testing], we will set up testing for each application and infrastructure component child module by creating a `test/` module within each module's directory. The test module will define any dependent resources required by the module, and then call the module itself. It could even call other application modules if needed.

### Testing Workflow

To use a module's `test/` module for testing:

1. On a new feature branch, make modifications to the functionality of the module
1. In that same branch, add test cases for the new functionality to the module's testing configuration in `test/`
1. Switch to the default branch (`master`/`main`), `cd` to the `test/` directory and run `terraform apply` to stand up the test resources
1. Switch to the feature branch and run `terraform plan`/`terraform apply` to see the impact of the changes on the pre-existing resources
1. When finished testing, run `terraform destroy` to tear down all test resources

This workflow is intended to be carried out manually at first, but we might be able to use GitHub Actions to automate it at some point.

#### Caveats for testing ECS applications

In order to be able to test ECS applications, we will need to figure out how to allow the ECS service to retrieve a container image for deployment. There are a few blockers to this that will need to be overcome:

- We'll need to override the ECR repo URL with the production AWS account's repo, and give the test account permission to pull images
- We'll need to address an issue where apps can't start until the first GitHub Actions deploy because the template task definition references `latest`:
  - Option 1: Add `latest` tag to all ECR images
  - Option 2: Allow overriding image tag in module with [`.tfvars` files][terraform-tfvars].

### Isolated Testing Environment

As noted in the [guide-level explanation][guide-module-testing], testing will happen in the separate `mbta-ctd-test` AWS account. This account will be accessed using "assume role" permissions that full access to manage required AWS resources. The assumed role won't have full administrator access, but we'll manage the list of permitted services via IAM policies.

The test AWS account will have some base resources predefined, such as VPCs and subnets, via the `aws-ctd-test-base` Terraform root module. This module will also define an output a `context` var that test modules can use to reference these shared resources.

## 3. Standardizing on Infrastructure Component Modules
[reference-standardization]: #3-standardizing-on-infrastructure-component-modules

As noted in the [guide-level explanation][guide-standardization], this is currently the most mature of the proposed improvements, in that the `ctd-ecs-app` and `ctd-rds-db` modules already exist and are seeing use. Aside from increasing adoption and carrying out the [testing workflow][guide-module-testing] and [module versioning][guide-module-versioning] recommendations, other improvements that will be beneficial include:

- Handling all use cases for each module, using vars to enable/disable features that aren't likely to be used everywhere
- Building in support for maintenance modes to minimize temporary configuration changes
- Standardizing on the use of the `context` variable

## 4. Module Versioning/Publishing
[reference-module-versioning]: #4-module-versioningpublishing

As noted in the [guide-level explanation][guide-module-versioning], module versioning and publishing is something we will set up for all infrastructure component modules, but which should be considered optional for application modules. See also the [corresponding subsection under rationale and alternatives][alternatives-module-versioning].

### How To Publish A Module

To set up a module for versioning and publishing, we'll need to:

- Move the module's configuration to a separate repo, for a few reasons:
  - It simplifies the workflow for tagging/publishing releases
  - It's required if publishing to the public Terraform registry
- Ensure that the repo name follows the convention `terraform-aws-<module-name>`, which is a requirement for both [Terraform's public registry][terraform-module-publishing] and [Scalr's private registry][scalr-private-registry]
- Set up a tagging/release workflow in GitHub Actions that publishes the module to Scalr's private module registry or the public Terraform registry (ideally as a reusable workflow)
- Configure any references in root modules to call the module from the repo or registry instead of a relative path to the module

### Application Modules In Application Repos?

For application modules, the question has been raised as to whether it might make sense to move the module configuration into the application repo so it lives alongside the codebase. The stated advantages to this approach are as follows:

- All(?) of the application's configuration would live within the same repo
- Open source repos would include the necessary infrastructure configuration to deploy the application

Unfortunately this approach is not as advantageous as it seems. Some of the drawbacks include:

- Configuration that varies by environment will still live in root modules in the devops repo, so there will always be some application configuration that lives outside the application repo
- If publishing to a private registry, the workflows for tagging and publishing will need to live in the application repo, as will the resulting tags
- If publishing to Terraform's public registry, Terraform requires the module to live in a wholly separate repo with a strict naming convention and structure, which precludes this approach

Given the drawbacks, storing application modules in application repos is not recommended at this time.

## 5. Automating Terraform
[reference-automation]: #5-automating-terraform

As noted in the [guide-level explanation][guide-automation], our goal for automating Terraform is to procure Scalr. But in order to make optimal use of Scalr for Terraform automation, we need to understand where it will fit into our proposed workflows, and how best to configure it to suit our needs.

Scalr has a [hierarchical model][scalr-hierarchy] that allows for multiple layers of configuration. Within our Scalr account we can have multiple _environments_, and each environment can have multiple _workspaces_. Each workspace corresponds to a Terraform root module.

[Scalr Workspaces][scalr-workspaces] can be set up in one of three ways:

1. **VCS (GitHub) Integrated**, meaning configuration is managed in GitHub and runs are triggered as part of the pull request workflow (though running `terraform plan` on the command line is also supported)
2. **CLI Driven**, meaning configuration is managed as local files and runs are triggered exclusively by engineers running `terraform` commands
3. **Module Driven**, meaning configuration is managed by sourcing child modules directly and runs are triggered exclusively via the Scalr UI

For CTD's workflows, we'll be focusing entirely on **VCS Integrated** and **CLI Driven** workspaces, since Module Driven workspaces don't really fit into the way we use Terraform.

### Team Environments &amp; Workspaces

In keeping with the proposal for [splitting development/staging root modules by team][guide-module-splitting], each team will have their own Scalr environment where all their workspaces will live:

- The team's root module workspace (GitHub-integrated), where their development and staging resources will be managed
- Module testing workspaces (CLI-driven), where application modules can be tested in isolation

The base root modules (e.g. the current `prod` and `global`) will live in the Infrastructure team's environment, where all engineers will have some level of access to view runs and propose changes.

### Proposed Workflow

To carry out an infrastructure configuration change under the various workflow changes proposed in this document, an engineer will:

1. Make changes to an application or infrastructure component module
2. [Test those changes in isolation][reference-module-testing]
3. Deploy those changes to their team's development/staging environment to verify their impact
4. Deploy those changes to production once they are tested and approved

Scalr will be an integral part of each step in this workflow, as detailed below.

#### Scalr Workflow for Module Testing
[scalr-workflow-module-testing]: #scalr-workflow-for-module-testing

Each testing module will have a CLI-driven Scalr workspace with a remote operation backend that runs all `terraform` commands via Scalr. `terraform` commands will be run manually by the engineer, or possibly as part of a GitHub Actions workflow. Scalr's remote operation backend means the actual work will be done remotely by Scalr, using [Scalr's AWS permissions][reference-permissions], and the `terraform` CLI will show the run output locally.

##### Continuous Integration

- Engineer creates or modifies module configuration locally
- Engineer pushes changes to GitHub branch
- GitHub Actions workflow runs:
  - `terraform validate`
  - `terraform fmt`
- Engineer (or GitHub Actions) tests module creation/destruction via Scalr (see workflow description in [Module Testing Workflow section][reference-module-testing])
- Engineer opens pull request

##### Code Review/Approval

- Review &amp; approval handled via GitHub PR
- Branch protection rules ensure that PR cannot be merged without approval

##### Applying Changes

- Engineer merges approved PR
- If module versioning is employed, GitHub Actions triggers workflow to publish new version of module to module registry

Once a new version of the module is published, the module version can be updated in the development/staging and production contexts using the workflows described below.

If module versioning is not employed, changes to the module will need to be applied immediately in any dependent root modules. In the development/staging context, this can be handled automatically by configuring [run triggers](https://docs.scalr.com/en/latest/workspaces.html#run-triggers) in the Scalr workspace. In the production context, however, there will be an extra layer of protection to ensure that changes don't impact production before they're ready to be applied there. This is described in more detail in the [production workflow section][scalr-workflow-production] below.

#### Scalr Workflow for Applying Changes In Development/Staging Context
[scalr-workflow-development-staging]: #scalr-workflow-for-applying-changes-in-developmentstaging-context

Each root module will have a GitHub-integrated Scalr workspace. Scalr will automatically run `terraform plan` when a pull request is opened, and attempt to automatically run `terraform apply` on merge. It will also be possible to run `terraform plan` locally using Scalr's remote operation backend, but not `terraform apply`; this is to ensure that GitHub acts as the source of truth for controlling Terraform state.

##### Continuous Integration &amp; Testing

- Engineer updates configuration in local files, or module version number if module versioning is in use
- Engineer pushes changes to GitHub branch
- GitHub Actions workflow runs:
  - `terraform validate`
  - `terraform fmt`
- Engineer opens pull request
- Scalr runs `terraform plan`

##### Code Review/Approval

- Review &amp; approval handled via GitHub PR
- Branch protection rules ensure that PR cannot be merged without approval

##### Applying Changes

- Engineer or reviewer merges approved PR
- Merge triggers Scalr run:
  - If plan includes changes to IAM resources, Scalr blocks apply to hold it for infrastructure team approval
  - Scalr runs `terraform apply`

#### Scalr Workflow for Applying Changes In Production Context
[scalr-workflow-production]: #scalr-workflow-for-applying-changes-in-production-context

Production will mirror the development/staging context, with a few notable differences:

1. For the time being, all resources will remain in the central `prod` module, which will be [renamed][root-module-naming-convention] to `aws-ctd-main-prod` and have its own GitHub-integrated Scalr workspace.
2. All calls to un-versioned modules will pinned to particular git hashes instead of relative paths, in order to protect the production context from changes that are made to un-versioned modules.

As before, Scalr will automatically run `terraform plan` when a pull request is opened, and attempt to automatically run `terraform apply` on merge.

It is a security requirement to have an extra layer of protection on production systems, that any changes to those system require explicit approval. CTD typically handles this requirement during the code review stage. For Terraform changes, we can use the same approach, provided that sufficient protections are in place to guarantee that changes can't circumvent the code review process. Those protections will include:

- [Branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) for the default branch of the devops repo
- [A `CODEOWNERS` file](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) listing the required reviewers for each CTD application's Terraform configuration file within the `aws-ctd-main-prod` (née `prod`) root module

The code owners for each application can include engineering team leads, team members, and/or the Infrastructure team.

##### Continuous Integration &amp; Testing

- Module changes are first [tested in isolation][scalr-workflow-module-testing] and [applied in the development/staging context][scalr-workflow-development-staging]
- Engineer updates module version or ref in `aws-ctd-main-prod` root module
- Engineer pushes changes to GitHub branch
- GitHub Actions workflow runs:
  - `terraform validate`
  - `terraform fmt`
- Engineer opens pull request
- Scalr runs `terraform plan`

##### Code Review/Approval

- Review &amp; approval handled via GitHub PR
- GitHub assigns required reviewers based on CODEOWNERS
- Branch protection rules ensure that PR cannot be merged without approval

##### Applying Changes

- Engineer or reviewer merges approved PR
- Merge triggers Scalr run:
  - If plan includes changes to IAM resources, Scalr blocks apply to hold it for infrastructure team approval
  - Scalr runs `terraform apply`


### Policies and Approvals

In order to ensure that sensitive changes&mdash;such as those which affect IAM resources&mdash;can't be made without being properly vetted, we'll define OPA policies in Scalr that will check for changes to sensitive resource types. For more detail on this topic, see the [Balancing User Permissions][reference-permissions] section below.

## 6. Balancing User Permissions
[reference-permissions]: #6-balancing-user-permissions

As noted in the [guide-level explanation][guide-permissions], our goal is to achieve an optimal balance of security and usability with Terraform and AWS. Achieving this goal effectively is somewhat dependent on our ability to delegate permissions to an automated system like Scalr.

### Scalr Delegated Permissions

With Scalr, permission to manage AWS resources will be split into two layers: the AWS layer and the Scalr layer.

#### AWS Permissions

- Each Scalr workspace will have all required access to manage AWS resources for each module
  - Each module testing workspace will have an IAM role granting access to manage resources in the `mbta-ctd-test` AWS environment
  - Each team workspace will have an IAM role granting it access to manage resources in CTD's production AWS environment
- Engineers won't need individual permissions/AWS credentials to run `terraform plan`/`terraform apply`

The IAM roles and policies for each Scalr workspace will also be managed in Terraform, and engineers will be able to propose changes as needed. However, the root modules defining IAM permissions won't be managed by Scalr. For security reasons, we'll need to include the following safeguards in Scalr's AWS IAM role permissions:

- Scalr will only be allowed to manage IAM roles and IAM role policies (direct-attached, not managed policies) BUT NOT its own role and role policy
- Some modules may need to be refactored for compatibility with this goal if they currently define IAM managed policies; those will need to be converted to inline policies

#### Scalr Permissions

In all Scalr workspaces, engineers will have full permission to trigger an apply in Scalr, except for runs that contain IAM changes, which will be held for approval by the Infrastructure team. The process of holding changes for approval will be accomplished by configuring OPA "soft blocks" to make sure Scalr doesn't allow changes that include IAM to be applied without infrastructure team approval.

Ideally the infrastructure team should be included as reviewers on any GitHub pull requests that contain IAM changes in order to sign off on those changes. However, there doesn't seem to be a good way to enforce this requirement automatically. We considered sequestering IAM changes in certain paths and using `CODEOWNERS` to designate required reviewers, but there's nothing to stop someone from circumventing the review requirement by defining IAM resources in configuration elsewhere.

What we've opted to do instead is block runs at the apply stage to require approval before changes are applied. This unfortunately leaves the possibility that a pull request with unapproved IAM changes could be merged but not applied. In that situation, the Infrastructure team will have the ability to reject the apply, but we'll then need to follow up separately to revert the PR.

We feel that blocking IAM changes at the apply stage is a necessary and acceptable trade-off to mitigate the possibility of using Terraform to achieve IAM privilege escalation. For more on the thinking that led to this decision, see the [corresponding subsection under rationale and alternatives][alternatives-permissions].

### Modules Excluded From Scalr

Some resource types are sensitive enough that they'll always have to be excluded from application modules; as noted above, these will be managed in separate "restricted" modules that will be kept out of Scalr for security reasons. A non-comprehensive list of resources that will be considered restricted:

- IAM users
- IAM groups
- IAM managed policies
- S3 buckets with restrictive bucket policies

### User Permissions Without Scalr

In lieu of having Scalr in place to act as an intermediary for managing permissions, engineers still won't have permission to apply changes that include IAM resources. We'll need to find a way to proactively flag changes with IAM for infrastructure approval. This path has not been investigated fully, in the hope that we'll be able to procure Scalr.

# Drawbacks
[drawbacks]: #drawbacks

<!--
Why should we *not* do this?
-->

Most of the drawbacks of each specific recommendation are included inline. In brief:

- [Module versioning and publishing][guide-module-versioning] has some overhead that limits its utility
- Our ability to [procure Scalr as an automation solution][guide-automation] is not guaranteed (see [the relevant subsection in rationale and alternatives][alternatives-automation] for more on this)
- [Balancing permissions][reference-permissions] effectively is hard

Some additional drawbacks are outlined below.

## This Is A Lot

The biggest drawback to this proposal is that it represents an enormous amount of work. Achieving all of the goals set out in this RFC is going to require sustained effort from, and coordination between, the infrastructure and engineering teams. This certainly isn't a reason not to move forward, but it's worth noting that this project will take time, and will need to be prioritized appropriately in order to maximize the positive impacts to the team.

## Scalr Caveats
[drawbacks-scalr]: #scalr-caveats

Potential drawbacks to using Scalr for Terraform automation:

- In the short term, migrating to Scalr means moving our state from S3 into Scalr (supporting S3 state is on their roadmap)
- We need to maintain a custom set of permissions ("roles") in Scalr to support the [approval workflow for OPA policies][reference-permissions]
- We won't have access to Scalr's [custom hooks][scalr-custom-hooks] feature without paying for a higher-tier plan (but we should be able to use GitHub Actions to meet most of our needs)

For potential alternatives to Scalr, see the [corresponding subsection under rationale and alternatives][alternatives-automation].

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

This list may not be comprehensive; feel free to suggest additional alternatives that aren't covered here.

## Alternatives to Splitting Root Modules
[alternatives-module-splitting]: #alternatives-to-splitting-root-modules

We feel the [proposed approach][guide-module-splitting] adds a lot of flexibility without adding more complexity than the team can withstand. Splitting by team will allow us to manage Terraform (and Scalr) permissions at the team level, which will be significantly easier than splitting in other ways. Focusing on splitting the `dev` module first will help us iron out the process and ensure that the resulting workflows are sensible before applying them to production in a future migration.

One alternative approach would be to split modules by individual application instead of by team, but that would add complexity without adding much more in the way of flexibility, and is likely more granularity than we need at this point.

## Alternatives to Module Testing Workflow
[alternatives-module-testing]: #alternatives-to-module-testing-workflow

The [Proposed approach][guide-module-testing] is relatively easy to implement for each module. The bulk of the work required will be to author each testing module.

[Terratest](https://terratest.gruntwork.io/) is an alternative testing framework for Terraform, but it requires writing tests in Go, which is a language that CTD has relatively little experience with, and the Infrastructure team in particular has none.

HashiCorp has documented their own [Module Testing Experiment](https://developer.hashicorp.com/terraform/language/modules/testing-experiment) that's similar in principle to our proposed workflow. It implements a `terraform test` command that carries out a testing workflow similar to [the one proposed in this document][reference-module-testing]. The documentation is explicit about their workflow being experimental and likely to change in the future, so it's not yet mature enough for our needs. If and when it does become a more established feature, it should be relatively easy to layer it on top of our proposed workflow.

## Alternatives to Standardizing Infrastructure Components
[alternatives-standardization]: #alternatives-to-standardizing-infrastructure-components

The [Proposed approach][guide-standardization] is already mostly implemented. It uses modules that we have mostly written in-house, though some components were copied from modules published in [Terraform's public registry](https://registry.terraform.io/).

We could switch to using published modules exclusively, or at least prefer them over the versions we maintain locally. This option was evaluated early on in the original Terraform implementation, but at the time it was unclear how stable each published module would be in the long term, and we were uncomfortable relying on public modules for critical infrastructure configuration. We feel that maintaining our own infrastructure components gives us more flexibility to configure modules how we need to.

## Alternatives to Module Versioning and Publishing
[alternatives-module-versioning]: #alternatives-to-module-versioning-and-publishing

The [Proposed approach][guide-module-versioning] aligns us to a [Terraform best practice][terraform-module-publishing], which is arguably beneficial even if we don't publish our modules publicly.

### Alternative 1: Do Nothing

The no-build option would be to keep storing modules in the devops repo and calling them using relative paths. This is the current recommendation for application modules. The drawbacks to this approach are:

  - We must rely on feature flag variables to prevent new functionality from being applied immediately everywhere the module is used
  - It doesn't scale well for modules that are used frequently

### Alternative 2: Monorepo for Private Modules

Another alternative would be to move all modules that aren't destined to be published in Terraform's public registry into a separate monorepo. This method is supported by Scalr, provided that the folder naming convention matches the official convention of `terraform-aws-<module_name>`. One advantage is that it would be easier to manage the versioning and publishing workflow for private modules in one place. One potential disadvantage is that it could be confusing and awkward to have public modules in separate, individual repos and private modules in a separate monorepo. It would also require additional work to migrate all modules, for what is arguably limited benefit. For these reasons, we opted to recommend keeping un-versioned application modules in the devops repo and versioned modules in separate repos.

### Alternative 3: Use Git Hashes

One other alternative that amounts to a compromise option is to keep modules un-versioned, but call them using a git hash instead of a relative path. This approach is easier to implement, since it doesn't require a tagging workflow for modules, yet still yields the benefits of versioning in that a module's configuration can be changed without immediately impacting all the contexts where it's used. The main disadvantage is that git hashes are harder to read than version strings, so it's not as clear what version of a particular module is in use. This option is being recommended as a [short-term solution for calling application modules in production][scalr-workflow-production], as a way to pin module calls without the hassle of a versioning/publishing workflow.

## Alternative Automation Solutions
[alternatives-automation]: #alternative-automation-solutions

### Alternatives to Scalr

In addition to evaluating Scalr, we also looked at env0, Spacelift, Atlantis, and HashiCorp's own Terraform Cloud.

**[Terraform Cloud](https://cloud.hashicorp.com/products/terraform)** was the least flexible of the options we evaluated, and it didn't feel like it added a lot of value to our workflow. The lack of ability to use the Terraform CLI for plans in GitHub-driven workspaces was seen as a major shortcoming. It felt like HashiCorp wanted us to replace their command-line tool with their cloud platform, and we weren't ready to make that strong a commitment to their product.

**[Spacelift](https://spacelift.io/)** was compelling feature-wise, but was also the highest-cost option, and didn't offer much additional value for the price vs. their competitors.

**[env0](https://www.env0.com/)** was also compelling feature-wise, and more competitive on price, making it our runner-up.

**[Atlantis](https://www.runatlantis.io/)** is a self-hosted option, and could be implemented without having to procure anything, though it would take some administrative overhead to get running. It also offers only a limited feature set compared to the other paid services we evaluated.

Ultimately we determined that Scalr offered equivalent features to Spacelift and env0 at a lower price point. It also offers the added functionality of a remote operation backend that handles all runs within their platform while still offering engineers the maximum flexibility of being able to run `terraform` commands locally. Finally, it strikes a balance of paying for features that save us time vs. having to self-host something like Atlantis that only achieves some of our needs.

### Alternatives Given Budget Constraints

As noted in the [guide-level explanation][guide-automation], CTD's ability to procure Scalr is not yet guaranteed for FY23. If we can't pay for Scalr in the short term, there are some recommendations in this RFC that we'll be able to achieve without it:

- [Root module splitting][guide-module-splitting] and the new [root module naming convention][root-module-naming-convention]
- The [module testing workflow][guide-module-testing], albeit manually and with AWS permissions via assumed role
- [Standardized infrastructure component modules][guide-standardization]
- Limited [module versioning and publishing][guide-module-versioning], restricted to Terraform's public registry

Additionally, there are some recommendations that we may be able to achieve with different solutions:

- Atlantis would give us semi-automated plan/apply, though it's unclear how we would effectively manage the AWS permissions used by Atlantis across multiple teams; it might require running multiple instances of Atlantis. It also offers policy checks with [conftest](https://www.runatlantis.io/docs/policy-checking.html) that could achieve our goals around IAM policy oversight.
- GitHub will play a role in versioning modules regardless of whether we have a private registry to publish them to, and we can reference git hashes in module calls if needed, as noted in the [module versioning alternatives section][alternatives-module-versioning].

However, there are some recommendations that we won't be able to achieve without Scalr:

- Granular per-user or per-team permissions to approve applies
- Queueing plan/apply runs to prevent state lock failures

There's a chance that we could achieve some level of Terraform command queueing via GitHub Actions, but it's unclear how well this will work, if at all.

### Scalr vs. Atlantis

If a free, self-hosted option exists for managing Terraform runs, why pay for something like Scalr? Essentially because:

- A software-as-a-service product will have less setup overhead than a self-hosted solution, which will save us time
- Atlantis lacks a user permissions system, which is something we intend to rely heavily on with Scalr
- Scalr includes a private module registry, for which there is no equivalent feature in Atlantis

### Policy Checks

One of Scalr's features is the ability to define policies that can intercept certain types of changes and require administrative approval. This feature factors into the proposed [Scalr workflows][scalr-workflow-production], particularly around protecting IAM resources. There are other solutions that could achieve this goal independently, namely [Open Policy Agent][open-policy-agent] and [Conftest][conftest]. Both are lower-level solutions that would require a lot of overhead to configure, so we're better off paying for this functionality from a vendor like Scalr that has already integrated it into their platform.

## Alternatives to User Permissions Structure
[alternatives-permissions]: #alternatives-to-user-permissions-structure

While working to understand the implications of [handing management of AWS resources over to an automated system][reference-permissions], we put some thought into different options for how to structure the management of IAM permissions.

Under our current Terraform configuration structure, application modules manage IAM roles and policies required by the application. If we allow engineers to manage IAM resources in an unrestricted manner, then in a worst-case scenario, an engineer could potentially add permissions to an application's IAM role that allows malicious code to be run or privileges to be escalated in a way that could be dangerous to our AWS account.

To mitigate this risk, three different models were discussed for managing IAM roles for applications. Model B is the current preferred approach.

### Permissions Model A

**IAM roles and policies remain in application modules, and individual users are given permission to maintain them**

#### Module Authoring & Testing

- All engineers will require IAM permissions in the AWS account used for testing Terraform modules

#### Dev Deployment

- All engineers will require IAM permissions in the AWS account where dev resources are deployed/hosted

#### Prod Deployment

- All engineers will require IAM permissions in the account where prod resources are deployed/hosted

#### Risks

- All engineers have the ability to apply IAM changes that can achieve privilege escalation, including outside of Terraform (i.e. directly via the AWS console or API)

### Permissions Model B

**IAM roles and policies remain in application modules, and an automated system is given permissions to manage them on users' behalves**

#### Module Authoring & Testing

- Engineers do not require IAM permissions in the AWS account where Terraform modules are tested
- Automated system requires IAM permissions in the AWS account where Terraform modules are tested

#### Dev Deployment

- Engineers do not require IAM permissions in the AWS account where dev resources are deployed/hosted
- Automated system requires IAM permissions in the AWS account where dev resources are deployed/hosted

#### Prod Deployment

- Engineers do not require IAM permissions in the AWS account where prod resources are deployed/hosted
- Automated system requires IAM permissions in the AWS account where prod resources are deployed/hosted

#### Risks

- Automated system has ability to apply IAM changes that can achieve privilege escalation
- **Mitigation:** Enforce policy checks that require approval from infra team if IAM resources are created/modified

### Permissions Model C

**IAM roles and policies are moved to separate modules and managed by designated administrators (infrastructure + team leads?)**

#### Module Authoring & Testing

- Testing modules requires two module calls – one to create IAM resources and one to create other resources
- Whoever is running tests (user or automation) will still require IAM permissions in order to test modules

#### Dev Deployment

- Dev deployment requires prerequisite deployment of IAM resource module
- Engineers do not require IAM permissions to deploy changes to dev

#### Prod Deployment

- Prod deployment requires prerequisite deployment of IAM resource module
- Engineers do not require IAM permissions to deploy changes to prod

#### Risks

- Onerous process to deploy IAM modules separately from application modules
- Module testing requires creating IAM resources separately
- Engineers can still define IAM resources anywhere, though they'll lack the permissions to apply them

# Prior art
[prior-art]: #prior-art

<!--
Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what
this can include are:

- Can we learn something about this proposal from other projects in CTD?
- Can we learn something about this proposal from other transit agencies?
- Can we learn something about this proposal from past experiences in other jobs or projects?

This section is intended to encourage you as an author to think about the lessons from other places,
and provide readers of your RFC with a fuller picture.

If there is no prior art, that is fine.

Note that while precedent is some motivation, it does not on its own motivate an RFC.
-->

Workflow automation for Terraform is entirely new territory for the authors of this document. We've done a fair amount of research and factfinding to formulate a plan that we think brings CTD's usage of Terraform more in line with industry best practices, and takes advantage of products that take some of the work off our shoulders. That said, it's entirely possible there's something we missed, and we're open to recommendations of solutions, software or patterns from anyone who cares to weigh in.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?
-->

One major unresolved question is the timeline for implementing all the improvements laid out in this RFC. As noted in the [Drawbacks][drawbacks] section above, it's a lot of work. Once this RFC is approved we'll need to scope out the work and divide it accordingly between the infrastructure and engineering teams.

Other questions for the reader:

- Is the proposed [root module naming convention][root-module-naming-convention] sensible and sufficiently comprehensive?
- Does the [isolated module testing workflow][reference-module-testing] make sense? Does it seem easy enough to carry out?
- How aggressive should we be about [standardizing on infrastructure component modules][guide-standardization] for ECS and RDS across all applications/teams?
- How aggressive should we be about [versioning and publishing application modules][guide-module-versioning] for open-source applications?
- If changes to non-versioned application modules can spill over into other modules, as noted in the [Scalr Workflow for Module Testing][scalr-workflow-module-testing], is that enough of a reason to push to version and publish all application modules?
- Does the decision to [delay splitting the `prod` module and instead version all production module calls with a git hash][scalr-workflow-production] make sense?

# Future possibilities
[future-possibilities]: #future-possibilities

<!--
Think about what the natural extension and evolution of your proposal would be and how it would
affect the project as a whole in a holistic way. Try to use this section as a tool to more fully
consider all possible interactions with the project in your proposal. Also consider how this all
fits into the roadmap for the project.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but
otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot
think of anything.

Note that having something written down in the future-possibilities section is not a reason to
accept the current or a future RFC; such notes should be in the section on motivation or rationale
in this or subsequent RFCs. The section merely provides additional information.
-->

Ideas that are out of scope for the work described in this RFC, but are potentially worth implementing later:

- Splitting production resources out of the `prod` module and into team-based root modules
- Handling the [module testing workflow][reference-module-testing] as a GitHub Actions workflow instead of manually
- Publishing more of our modules in [Terraform's public registry][terraform-module-publishing]
- Investigating [Terragrunt](https://terragrunt.gruntwork.io/) as a way to standardize backend and provider configuration and reduce redundant configuration

[conftest]: https://www.conftest.dev/
[open-policy-agent]: https://www.openpolicyagent.org/
[scalr-custom-hooks]: https://docs.scalr.com/en/latest/workspaces.html#custom-hooks
[scalr-hierarchy]: https://docs.scalr.com/en/latest/hierarchy.html
[scalr-private-registry]: https://docs.scalr.com/en/latest/module.html
[scalr-workspaces]: https://docs.scalr.com/en/latest/workspaces.html
[terraform-module-publishing]: https://developer.hashicorp.com/terraform/registry/modules/publish
[terraform-tfvars]: https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files
[terraform-workspaces]: https://developer.hashicorp.com/terraform/cli/workspaces
