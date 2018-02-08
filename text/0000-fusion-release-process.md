* Start Date: 2018-02-06
* RFC PR: (leave this empty)
* React Issue: (leave this empty)

# Summary

A better process for release quality verification and publishing for the FusionJS core ecosystem.

# Motivation

Currently FusionJS packages live within multiple repositories and it's hard to determine the state of the FusionJS core ecosystem. We often release breaking changes in one package, but don't get around to upgrading dependent packages, or are aware of cross-package breakage that occurs. Our lack of cross-repo verification often leads to a fragmented and broken ecosystem, with no signal as to the overall current state.

The goal of this RFC is to define a verification and release process, certifying FusionJS release quality, and allowing for faster and higher quality releases.

# Detailed design

### On-demand monorepo construction

We will continue to develop individual FusionJS plugins and libraries within individual repositories. This gives us fast development cycles and avoids monorepo pain points like long boostrap time. A new CI job will exist which will be triggered for every commit going into master across all repositories. This job will construct a monorepo and run tests against the latest HEAD for all FusionJS repositories.

We will have the best of a multirepo world, along with tooling that gives us the benefits of a monorepo.

### Topologically sorted dependency graph

We build a topologically sorted, batched graph of dependencies which will be used to build modules in parallel. FusionJS plugins will be transpiled, and copied into every dependent module. Non-fusion dependencies will continued to be installed through yarn. We choose to copy modules rather than symlink as that's more representative to an end-user using FusionJS, and also works better with static analysis.

### Private package support

Due to containerization, this project should be able to support both open source and closed source projects given that there is sufficient infrastructure. For open source projects we plan on running this within Buildkite, verifying each package on one or more parallel workers. A list of additional private projects may be contained within a gitignored file, or pulled down at runtime from another source. All of the existing topological sorting and package linking should continue to work.

### Release verification

We will verify releases by triggering a verification pipeline which runs unit tests, linters, flow and integration tests across the HEAD of all packages. This pipeline would be run at a scheduled interval, as well as every commit to master across all FusionJS repositories. Only by running on HEAD can we discern whether or not we are safe to release or not. Integrations via email, uchat and dashboards will notify us of build status and breakages.

### Release lockstep publishing

We will stop publishing packages individually, and instead lockstep the entire release, leveraging CI for publishing. Assuming that all tests pass, we can kick off a pipeline job which will use our built monorepo to publish packages. This will save engineers time as they don't need to open pull requests to every repository to publish. One possible flow of how this would work:

* An engineer manually kicks off a Buildkite pipeline to publish.
* The engineer is prompted to choose the next version using [Buildkite input fields](https://buildkite.com/docs/pipelines/block-step#select-input-attributes), but a default version is provided as well.
* Verification is performed on the release version, doing what we can to ensure that breaking changes enforce the next major version bump.
* When the user presses the build button, we run all tests. If the tests fail the pipeline will not be allowed to perform a version publish.
* The pipeline builds a topologically sorted list of packages. For each batch of packages the pipeline will open a pull request against the repo with the package bump.
* The pipeline will monitor status, and automatically land the version bump when tests pass.
* Once the packages are available in the NPM registry we continue down the list of sorted packages, until everything has been published.

# Drawbacks

### Complexity

There's a fair amount of complexity involved in our current fusion-release repository, and it will likely grow over time. Pieces that add complexity include usage of a topological sorting library, as well as a pipeline which defines the entire testing pipeline. This pipeline is currently not generated due to slight differences between repos, and will need to be updated as we add additional plugins and test suites. One alternative is creating a yarn script for each plugin which will allow us to shell out to that command per repository. E.g., yarn run release-verification.

### Build time

Bootstraping a monorepo from multiple repositories can take a long time. We currently spend about 30 minutes bootstrapping and uploading a 5GB docker image. Luckily individual repo CI is still fast and can complete within a matter of minutes. There may be some ways to improve this bootstrap time, but that needs to be investigated.

# Alternatives

* Keep doing what we're doing today.

# Adoption strategy

### How frequently do releases happen?

First and foremost we should strive to keep master stable, and releasable at any time. Whether that's a patch release that we decide to release every few days, or a breaking change with codemods, we should always be able to ship when needed. This will keep us responsive and able to fix bugs in a timely manner without introducing new bugs.

Outside of our releass we will also plan on having milestones for features, and perform a release when that milestone is done. With comprehensive testing and diligence around writing codemods, we should generally be able to avoid the case where a breaking change in master would prevent us from shipping a new version. In the case this does happen, branching off at an earlier release for an individual package is always an option.

### Impact of open source contributions against private repositories

We will always conduct ourselves in a way to do what is best for the open source codebase. If a breaking change that impacts internal code would be best for the framework, we should accept it, assuming that the code meets our guidelines and codemod requirements.

### How does greenkeeping propagate to existing consumers of FusionJS

Greenkeeping will be painless due to an improved focus on release quality, semvar adherence and codemod diligence. We will provide a CLI tool to automatically upgrade FusionJS packages and run the associated codemods for that upgrade. Codemods should be placed in a folder for each major release, and should ideally live within each individual repository.

Codemods directory specification:

```
fusion-core/
└── codemods/
    ├── 1.0.0/
    |   └── change-token-name.js
    └── 2.0.0/
        └── some-other-codemod.js
```

# Unresolved questions

* Please add commenmts with any questions.
