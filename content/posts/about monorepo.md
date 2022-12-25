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

I must confess after 1.5 year after I joined, we are still using multiple-repositories. But things have been largely improved to gear towards a mono-repository.

What I've setup is a gitops strategy:
* all repositories are building on their own and generates artifacts: Docker images or zip archive (we have part of our infrastructure that's running on-premise in the field). Artifacts are archived and tracked using a git tag or branch name
* we are using Kubernetes with full automation
* we are using the expensive GitHub Enterprise to only use environment with protection rules (21$ for this is crazy but no choice for the moment)
* everything is deployed using Terraform with strict versionning
* all deployments happens using GitHub actions within protected environments only
* we have more deployment piece since splitting the big server-side monolith was a requirement to move faster across teams (yeah, we went for micro-services using [fbus](https://github.com/pchalamet/fbus))

Basically, this means we have more repositories than before 🤪.
But we are in good shape to switch to mono-repository now 🎉. Everything is running quite smooth. So why move from that model?

Well, feature development is a pain in the neck. Most of our development implies impacting several applications from front-end to back-end while changing database models from time to time. This is difficult for most devs to grasp it well:
* isolate from main branch (especially when several repositories are impacted) the time feature stabilizes
* understand impacts for testing
* understand impacts for a release to ensure smooth communication with support team
* do not miss something when deploying to test environment (we have micro-services again)
* get the correct merge period to lower impacts

From an operational point of view, there are also several things hard to track:
* reviews are complicated: usually this spans several repositories and it's a hell to understand what's going on
* tests are underestimated (due to incremental impacts)
* communication is impaired has several repositories have to be tracked
* devs are not motivated with large changes/refactoring

All in all, I'm a strong advocate for mono-repository for just one thing: atomic feature implementation.

But if you think going mono-repository is easy, you are totally wrong. Single app per repository (aka multiple-repositories) is easy: setup sources, build and generate artifact on changes. So easy to do and no performance problem.
When using mono-repository, you will hit the wall for sure: time to build the applications and noise. And most of the time, there is no need to rebuild everything. We only need to rebuild and release what has changed. Shipping the whole platform is a non-sense. Noise is a clear problem has looking at history is not so funny. Hopefully, Meta has release a tool to understand what's going on [Sapling](https://github.com/facebook/sapling). I've not tested it, but maybe this can help inthe future.

So what to do ? Well, you mono-repo must have tools to help optimizing the build:
* identify what has changed (a new commit, a new branch)
* build only what has changed - and think about changes in libraries up to deployment
* delivery what has changed - generate a release note for changes

A mono-repository requires much more work to setup than multi-repositories. But benefits are tremendous.

I discover mono-repository (or at least unified view of mult-repositories) at [Criteo](https://criteo.com): that was the MOAB (Mother Of All Builds). When I left Criteo, I've decided to create [Full-Build](https://full-build.io) to help D-Edge move faster in engineering side. Full-Build is no much maintained publicly speaking but it lives under various names today (no more public). I've considered doing a v2 as I'm not really satisfied with current offer on the market:
* [Bazel](https://bazel.build) (Google) / [Buckbuild](https://buck.build) (Meta): probably the top offer to consider but requires dedicated teams - does not really fit the startup/mid-company. Lack reuse of existing project metadata. 
* Turborepo: javascript only
* NX Build: nice plugin support but javascript builds are so messy. Also, does not provide proper isolation, this leaks everywhere. It works ok with lots of sweats.

What I'm looking for:
* be explicit with projects declaration: no magic
* use most of metadata from projects (npm, .net projects, maven...) to get dependencies
* rely on eco-systems build instead of relying on plugins
* ensure strong projects isolation - all paths must be relative to project, not workspace
* support for explicit tasks

All in all, I think I will go for full-build v2 😃 I just need a tool that is no brainer and definitively open-source.