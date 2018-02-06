* Start Date: 2018-02-06
* RFC PR: (leave this empty)
* React Issue: (leave this empty)

# Summary

A better process for release quality verification and publishing for the FusionJS core ecosystem.

# Motivation

Currently FusionJS packages live within multiple repositories and it's hard to tell the state of the FusionJS core ecosystem. We often release breaking changes in one package, but don't get around to upgrading dependent packages, or are aware of cross-package breakage that occurs. Our lack of cross-repo verification often leads to a fragmented and broken ecosystem, with no signal as to the overall current state.

The goal of this RFC is to define a verification and release process, certifying FusionJS release quality, and allowing for faster and higher quality releases.

# Detailed design

### On-demand monorepo construction

We will continue to develop individual FusionJS plugins and libraries in individual repositories. This gives us fast development cycles and avoids monorepo pain points like long boostrap time. A new CI job will exist which will be triggered for every commit going into master across all repositories. This job will construct a monorepo and run tests against the latest HEAD for all FusionJS repositories.

We will have the best of a multirepo world, along with tooling that gives us the benefits of a monorepo.

### Topologically sorted dependency graph

We build a topologically sorted, batched graph of dependencies which will be used to build modules in parallel. FusionJS plugins will be transpiled, and copied into every dependent module. Non-fusion dependencies will continued to be installed through yarn. We choose to copy modules rather than symlink as that's more representative to an end-user using FusionJS, and also works better with static analysis.

### Private package support

Due to containerization, this project should be able to support both open source and closed source projects given that there is sufficient infrastructure. For open source projects we plan on running this within Buildkite, verifying each package on one or more parallel workers. A list of additional private projects may be contained within a gitignored file, or pulled down at runtime from another source. All of the existing topological sorting and package linking should continue to work.

### Release verification

We will verify releases by running unit tests, linters, flow and integration tests across the HEAD of all packages. Only by running on HEAD can we discern whether or not we are safe to release or not.

### Release lockstep publishing

Assuming that all tests pass, we can kick off a pipeline job which will use our built monorepo to publish to the NPM registry. We can leverage a Buildkite input or select dropdown to choose the next version. (https://buildkite.com/docs/pipelines/block-step#select-input-attributes)

Even though we did not bootstrap with Lerna, we can take advantage of their publishing scripts to lockstep-publish all FusionJS repositories. If we do publish from this pipeline, we would need the repository to commit both package.json and yarn.lock to each repository on Github.

# Drawbacks

### Complexity

There's a fair amount of complexity involved in our current fusion-release repository, and it will likely grow over time. Pieces that add complexity include usage of a topological sorting library, as well as a pipeline which defines the entire testing pipeline. This pipeline is currently not generated due to slight differences between repos, and will need to be updated as we add additional plugins and test suites. One alternative is creating a yarn script for each plugin which will allow us to shell out to that command per repository. E.g., yarn run release-verification.

### Machine cost

Bundling FusionJS as a monorepo results in a 5GB docker image along with dozens of hours of machine time. Compared to engineering cost the price is miniscule, but there is a cost to running all of these compute hours on AWS (50 machine hours being $1 per commit to test on spot instances or so). We would ideally receive a budget for something like "500 compute hours per pull request" and not have to worry about cost at all.

# Alternatives

* Keep doing what we're doing today.

# Adoption strategy

### Release version lockstepping

We would stop publishing packages individually, and instead lockstep the entire release. This would likely save developers time as they would not need to manually version bump.

### Verification pipeline kickoff

We would trigger a verification pipeline job to be run at a scheduled interval, or every commit to master. We should also add integration so we are notified of breakages.

# How we teach this

Document our release processes.

# Unresolved questions

* Please add commenmts with any questions.
