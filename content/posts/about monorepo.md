---
title: "About monorepo"
date: 2022-12-25T9:00:00+01:00
draft: false
---

It's been something like a year and half I've been CTO at [Tessan](https://tessan.io). When I joined, I've found the same problems that plague engineering teams:
* lack of delivery practices
* ship branches instead of versions (using kind of gitflow - that was soooo scary)
* no testing but manual testing
* no metrics in production
* no cadence to deliver feature
* no way to enable a feature in production once mature
* manual deployments

As most company, Tessan is using a multi-repositories strategy. Like a vast majority of companies, this is just the situation that has been reached without much thoughts: start with a project, start a second, discover builds are complicated - split into 2nd repository,... rince again That's where most companies are. No strategy, no thinking about what can be done to improve things or improve delivery.

I must confess, we are still using multiple-repositories. But things have been largely improved to gear towards a mono-repository.

What has been implemented is a gitops strategy:
* repositories have dedicated build and generates their own artifacts: Docker images or zip archive (we have part of our infrastructure that's running on-premise in the field). Artifacts are archived and tracked using a git tag or branch name
* everything is deployed using Terraform with strict versionning. Targets are Kubernetes and on-premise infrastructure
* all deployments happens using GitHub actions within protected environments hence we are using the super expensive GitHub Enterprise just to only use protection rules (this badly hurts)
* we have more deployment pieces since splitting the big server-side monolith was a requirement to move faster across teams (yes, we went for micro-services using [fbus](https://github.com/pchalamet/fbus))

Basically, this means we have more repositories than before 🤪.
But we are in good shape to switch to mono-repository now. Everything is running quite smooth. So why move away from that model?

Well, feature development is a pain in the neck. Most of our development implies modifying several applications from front-end to back-end while changing database models from time to time. This is difficult for most devs to grasp:
* isolate from main branch (especially when several repositories are impacted) the time feature stabilizes
* understand impacts for testing
* understand impacts for a release to ensure smooth communication with support team
* do not miss something when deploying to test environment (we have micro-services again)
* get the correct merge period to lower impacts

From an operational point of view, there are also several things hard to track:
* reviews are complicated: usually this spans several repositories and it's a hell to understand what's going on
* tests are underestimated (due to incremental impacts)
* communication is impaired has several repositories have to be tracked

And from a dev perspective, it's rather not good:
* no motivation for changes/refactoring across repositories
* lack of understanding how things keep working despite partially released (nullables, feature flags...)
* no opportunity to learn by reading *more* code

For at least one thing, I'm a strong advocate for mono-repository: atomic feature implementation.

But if you think going mono-repository is easy, you are totally wrong. Single app per repository (aka multiple-repositories) is easy to do: setup sources, build and generate artifact on changes. Done.

When using mono-repository, you will hit the wall for sure: time to build the applications and noise:
* Most of the time, there is no need to rebuild everything. We only need to rebuild and release what has changed. Shipping the whole platform is a non-sense.
* Noise is a clear problem has looking at history is not so funny. Hopefully, Meta has release a tool to understand what's going on [Sapling](https://github.com/facebook/sapling). I've not tested it, but maybe this can help inthe future. To be investigated.

So what to do ? Well, your mono-repo must have tools to ensure it's fast, optimize the build and provide auditing features:
* identify what has changed (a new commit, a new branch)
* build only what has changed - and think about changes in libraries up to deployment
* delivery what has changed - generate a release note for changes

A mono-repository requires much more work to setup than multi-repositories. But benefits are tremendous.

I learned about mono-repository (or at least unified view of mult-repositories) at [Criteo](https://criteo.com): that was the MOAB (Mother Of All Builds). When I left, I decided to create [Full-Build](https://full-build.io) to help D-Edge move faster engineering side. Full-Build is not much maintained (publicly speaking) but it lives under various names today (no more public). I've considered doing a public v2 as I'm not really satisfied with current state of affair.

Anyway, there are several tools on the market:
* [Bazel](https://bazel.build) (Google) / [Buckbuild](https://buck.build) (Meta): probably the top offer to consider but requires dedicated teams - does not really fit the startup/mid-company. Lack reuse of existing project metadata
* Turborepo: javascript only
* NX Build: nice plugin support but javascript builds are so messy. Also, does not provide proper isolation, this leaks everywhere. It works ok with lots of sweats

But as I'm saying, I'm not really satisfied. What I'm looking for:
* be explicit with projects declaration: no magic
* use most of metadata from projects (npm, .net projects, maven...) to get dependencies
* leverage eco-systems instead of relying on plugins
* ensure strong projects isolation - all paths must be relative to project, not workspace
* support for explicit tasks

All in all, I think I will go for full-build v2 😃 I just need a tool that is no brainer and definitively open-source.