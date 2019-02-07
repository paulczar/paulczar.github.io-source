---
date: "2019-01-29"
title: "Cloud Native Operations - 12 Factor Infrastructure"
categories: ["kubernetes","pivotal","devops", "cloud-native"]
draft: true
---

# Introduction

Cloud Native Infrastructure is taking best practices and techniques for developing Cloud Native Applications and applying them to the Infrastructure on which those Applications run.

One of the seminal pieces of literature for developing Microservces and Cloud Native Applications in general is the [12 Factor Application Manifest](https://12factor.net/). It provides 12 fairly easy to understand concepts that if followed will ensure that your application is designed to run on a cloud platform such as Cloud Foundry or Heroku.

These 12 factors apply really well to Infrastructure concepts as well, and so lets create the __12 Factor Infrastructure Manifest__ to go with it.

# I. Codebase

A 12 factor infrastructure as always tracked in version control system such as Git. Generally speaking their are two sets of artifacts; Manifests and Inventories.

* __Manifests__ describe the resources required to build your infrastructure. A manifest is usually written in a DSL that is consumed by a specific tool such as Terraform, Ansible, or Helm.

* __Inventories__ describe the deployment of your infrastructure. They contain a series of variables that when combined with your _Manifest_ will deploy, update, or verify an environment (an environment is a deployed infrastructure).

Generally each Manifest would have its own code repository and a shared repository for the Inventories. Any secrets or keys should be encrypted before adding to the repo, or should be stored in an external encrypted secret store such as Hashicorp's Vault.

# II. Dependencies
Explicitly declare and isolate dependencies

# III. Config
Store config in the environment

# IV. Backing services
Treat backing services as attached resources

# V. Build, release, run
Strictly separate build and run stages

# VI. Processes
Execute the app as one or more stateless processes

# VII. Port binding
Export services via port binding

# VIII. Concurrency
Scale out via the process model

# IX. Disposability
Maximize robustness with fast startup and graceful shutdown

# X. Dev/prod parity
Keep development, staging, and production as similar as possible

# XI. Logs
Treat logs as event streams

# XII. Admin processes
Run admin/management tasks as one-off processes