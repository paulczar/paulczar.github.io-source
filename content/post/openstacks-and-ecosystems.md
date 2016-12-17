---
date: "2016-01-02T13:00:42-06:00"
title: "Openstacks and Ecosystems"
categories: [ "openstack" ]
---

I have recently had a number of lengthy discussions on the [Twitter](https://twitter.com/zehicle/status/678736665792356352) about Interop, Users, and Ecosystems. Specifically about our need to focus on the OpenStack ecosystem to extend the OpenStack IaaS user experience to something a bit more platform[ish].

I wrote a post for [SysAdvent](http://sysadvent.blogspot.com/2015/12/day-16-merry-paasmas-and-very.html) this year on developing applications on top of OpenStack using a collection of OpenSource tools to create a PaaS and CI/CD pipelines. I think it turned out quite well and really helped reinforce my beliefs on the subject.

My buddy and future OpenStack Board member [JJ Asghar](https://twitter.com/jjasghar) has been spearheading a new [OpenStack Operators Project](https://wiki.openstack.org/wiki/Osops). I plan to contribute to this project by creating some examples of deploying tools that provide higher level services on top of the OpenStack IaaS layer.

Given that I am very bullish about the [Docker](http://docker.com) ecosystem it makes sense that my first contribution would be focussed on running one of the several "Docker container scheduling/cluster" tools. 

After playing around with a few of them, I settled on starting with Docker Swarm as its one of the easier to understand and run and doesn't require any special tooling other than a recent install of the Docker binary to use.

To increase simplicity I chose to use Hashicorp's [Terraform](http://terraform.io) and use only the most basic of the OpenStack services to ensure a fairly high likelyhood that it will run on most fairly up to date OpenStack clouds.

Based on the project's suggestion I posted the Terraform files up to the [osops-tools-contrib](https://github.com/openstack/osops-tools-contrib/tree/master/terraform/dockerswarm-coreos) along with fairly comprehensive documentation on using it.

I hope this and future work I plan to do to create similar examples will help the OpenStack Community out in some small way.


