---
date: "2019-01-29"
title: "What is Cloud Native Operations?"
categories: ["kubernetes","pivotal","devops", "cloud-native"]
draft: true
---

# Lulz what? Cloud Native Operations ?!?!?!

In short Cloud Native Operations (CNO) is taking the ideas and concepts that were born out of Cloud Native Software Development such as microservices and the 12 factor manifest and applying them back to Operations practices such as devops and Site Reliability Engineering (SRE).

Cloud Native Operations does not supplant or replace devops or SRE but instead helps improve them. If I was asked to describe three core tenets of CNO I would give them as; Human, Composable and Observable.

# Human

The most important part of Cloud Native Operations is the People and therefore it should be optimized for them. Rather than Infrastructure as Code you should think Infrastructure as Configuration. Complex and hard to read things like Cloud Formations should be passed over. Rather look for tools that take simple configuration files or ordered instructions and act on those.

Tools like Hashicorp's Terraform or Ansible that provide a fairly simple DSL for defining resources are perfect, however do not be tempted to add much (if any) programming logic inside of them. The programming concepts around DRY (Don't Repeat Yourself) are great when you're writing complex low level libraries for reuse but can quickly make a Terraform manifest unreadable to anyone but the original author.

Most tools in this area have some form of DSL for describing your architecture and then some form of Inventory or set of values to be applied to that DSL at runtime. Ensure the variables used in your DSL that are exposed to the Inventory are very descriptive. A fellow Operator, even a level 1 support, should be able to view the inventory file and understand exactly what is being deployed and how.

The same goes for Configuration Management, often its seen as a point of accomplishment to write complex loops and other programming logic into a Chef Recipe even obfiscating large swathes of logic behind a lightweight resource provider. Avoid doing this wherever possible, like above it can reduce readability and increase likelihood of error.

Generally speaking you'll have a single artifact be it a Terraform manifest an Ansible Playbook, or even a Helm Chart that describes your architecture. You will have some form of Inventory artifact for each deployed environment. When you combine the two at deploy time you get a running environment. These artifacts should be stored in Version Control, with the latter having any secrets encrypted.

# Composable

# Observable