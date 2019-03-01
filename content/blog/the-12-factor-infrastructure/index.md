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

1. People

* People trump all other factors of Cloud Native Operations.
* Optimize for people first.

2. Platform [as a product]

* Ultimately you're building a platform, build it with a focus on the users and their needs.
* Use existing platforms and abstractions where possible and focus your efforts on smoothing out the rough edges and user experience
* Create the "paved road" that provides utility to your users and encourages them to use the platform by making it the _best choice_ for deploying their software.
* Do not treat it as an infrastructure project, treat it as a product, have a product owner to drive roadmap and features.
* Start simple and iterate. Do not go for "mass digital transformation" instead focus on one or two teams and help them solve their problems.

2. Everything As Code

* Automate everything with simple and clean code.
* Focus on readability and composability.
* Store in github and test

3. Composable

* The components of your infrastructure should be Composable and work together with external systems to create a coherent platform.
* Where possible use systems provided by the underlying platform

4. Observable

* Metrics and Alerts are important but only tell part of the story.
* Treat your logs as structured events
* Provide standard metric/logging APIs and endpoints for applications to tie into
* Provide tracing infrastructure to support complex systems of microservices

6. Configuration

* Provide a configuration service and service discovery
* Applications should be able to query one or more configuration servers to request their configuration
* Some form of encrypted secret management should be provided.

7. Security

* do not rely on edge security, every application should have its own security with explicit ingress and egress rules
* provide certificate and endpoint management

8. Pipelines
9. Artifacts

10. State
  Where-ever possible state should be kept outside of the application.

11. Resilience
12. Platform [as a product]
