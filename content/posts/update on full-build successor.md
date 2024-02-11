---
title: "Update on full-build successor"
date: 2024-02-11T18:28:37+01:00
draft: false
---

It's been a while I've been discussing about monorepo.\
Few things have changed: I've started to work on the successor of [full-build](https://full-build.io).

Why successor?

Well full-build was probably a step forward for source management but it's really a step backward in terms of compatibility: **modifying projects or dev workflows is bad**:
* Devs always forget to update those files
* Most of the time, this is different workflows on local and CI

In order to build a monorepo efficiently, it's important to take into account **how** devs are working:
* They use most of the time an IDE
* They use standard tools (make, Terraform, ...)
* They do not want to bother with stuff that interfer with their workflows

So OK, let's not change those habits and just:
* Use standard tools to build artifacts
* Never modify projects files
* Just provide tooling to build current branch and check if everything is ok

**Terrabuild** is then the successor of full-build. It's still being baked.\
Only few components are available at the [Magnus Opera](https://www.github.com/MagnusOpera) GitHub Organization.

It will allow building a monorepo using standard tools without any modifications to source. It will also allow to isolate builds and easily build with same toolchains as on CI. Terrabuild will use extensions to deal with various languages or tools and will support build caching for fast builds both local or on CI.

Stay tuned !