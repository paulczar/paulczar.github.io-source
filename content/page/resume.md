+++
date = "2020-02-09T09:04:08-06:00"
title = "Resume for Paul Czarkowski"
+++

# Paul Czarkowski

**Principal Managed OpenShift Black Belt @ Red Hat**

[username.taken@gmail.com](mailto:username.taken@gmail.com)

[http://tech.paulcz.net](http://tech.paulcz.net/)

[https://github.com/paulczar](https://github.com/paulczar)

Work Permits : USA, Australia, UK/Europe.

## Overview

I am an experienced hands-on Architect / DevOps Engineer who accidently fell into a Developer
Advocate role. I have a long history in Operations and Infrastructure Automation.  I have a broad depth of experience across most IT and Operations related areas with strong experience in using and evangelizing DevOps tools and methodologies.

My current team (Managed OpenShift Black Belts) at Red Hat is a post-sales techincal team focussed at removing technical and organizational blockers within customers to help them accellerate their usage of our cloud services. In doing this I have developed expertise in not just OpenShift, but the underlying clouds (Azure, AWS, Google) and any number of integrations with the native cloud services. You can see much of my public facing work at [mobb.ninja](https://mobb.ninja) showing these off.

Previously at at VMware I worked as a Developer Advocate. It was a customer and public facing role in which I worked to Advocate for customers and opensource communities into VMware, as well as Advocating for VMware to customers and communities. I als hado have a key role in improving the Operator experience for a number of VMware's flagship products. I was (and am still) heavily involved in the Kubernetes community and can be found all over the world advocating for Kubernetes and am an core member of the Helm and Helm Charts community.

My previous role at IBM has me focused on helping to rebuild the IBM developer advocacy programs and content creation processes.  Previous to that I led the conversion of our OpenStack Automation platform ([ursula](https://github.com/blueboxgroup/ursula)) from just Ubuntu to also supporting Redhat Enterprise Linux.  I also architected and built the Blue Box Cloud SRE Operations Platform (which was open-sourced as [cuttle](https://github.com/IBM/cuttle)) and built a team to maintain it.

Previous to IBM/Blue Box I was at Rackspace where I worked on a team building a product with Docker on top of Openstack, and before that I worked at EA where I helped build and design the infrastructure for SimCity ( on AWS ) and SWTOR ( own data centers, approx 6,000 servers, 2M+ subscribers at launch ).

## Speaking Engagements

As a Developer Advocate I have travelled to all corners of the globe to speak about Kubernetes, DevOps, and related topics. Some highlighted events, recordings, and slide decks can be found at my Speaking page: [https://speaking.paulcz.net/](https://speaking.paulcz.net/)

## Open Source and Passion Projects

* Built out a platform in a box project to deploy a full SRE/Ops stack of tools to turn Kubernetes into a fully features platform - [Platform Operations on Kubernetes](https://github.com/paulczar/platform-operations-on-kubernetes).

* Built the first viable [Google Cloud Operator]((https://github.com/paulczar/gcp-cloud-compute-operator)) for Kubernetes that let you manage most popular Google Cloud services from within Kubernetes, Worked with the google team to help improve their official operator "Config Connector" to reach (and later exceed) feature parity with mine.

* In an effort to reduce the toil involved in managing and deploying to Helm Repositories I wrote a tool called [Chart Releaser](https://github.com/helm/chart-releaser) which uses github + github pages to fully host helm chart repositories. This tool was adopted into the official Helm repository as an official project and is now used in production for hundreds of Helm Chart repositories.

* At IBM I involved myself in the Kargo/Kubespray community and made a significant amount of contributions to help improve the quality of the Ansible being written and the composability of the Roles. Surprisingly a year on I'm still in the top [5 contributors](https://github.com/kubernetes-incubator/kubespray/graphs/contributors) (based on lines of code, which is obviously the most important metric AMIRITE).

* Built out a Chef Inspec Repository for [RedHat 6 STIG auditing](https://github.com/inspec-stigs/inspec-stig-rhel6) and formed a small community around using [Inspec for STIG auditing](https://github.com/inspec-stigs).

* I got tired of fighting openssl commands to create SSL/TLS for development so I built a Docker Image called [omgwtfssl](https://github.com/paulczar/omgwtfssl) that takes a few environment variables and spits out a CA/key/cert combo.

* Over Christmas 2014 I built out [Factorish](https://github.com/factorish/factorish) as a concept to show managing the life-cycle and configuration of applications in Docker using service discovery, and built several example apps such as [Percona with Galera Replication](https://github.com/paulczar/docker-percona_galera) and the [ELK stack](https://github.com/factorish/factorish-elk).  Some of these concepts have found their way into tools such as [Container Pilot](https://github.com/joyent/containerpilot) and [Habitat.sh](https://habitat.sh).  I also used it as a basis for a [blog post](http://tech.paulcz.net/blog/factorish_and_the_12_fakter_app/) and a series of talks I gave on Dockerizing apps that really shouldn’t be Dockerized.

## Professional Accomplishments

### VMware / Pivotal

* Traveled to 20+ countries and presented talks, workshops, demos, to thousands of people, from customers, to community, to individual coaching, small meetups, to massive conferences such as KubeCon and VMWorld.

* Coached and mentored my (mostly developer) team in DevOps concepts and usage of Kubernetes and related technologies.

* Helped plan, organize, and develop our first major outreach initiative after the COVID related travel restrictions, resulting in a 24+ hour Spring Live conference on March 19th 2020 with tens of thousands of attendees.

* Took the learnings from Spring Live and lead the creation of a streaming practice [tanzu.tv](https://tanzu.tv) for developer outreach with 5 weekly shows and more coming. Became the team expert in creating video content for Twitch, Youtube, both streaming, as well as more traditional content.

### IBM / Blue Box

* Led the effort to port the Blue Box OpenStack automation tool ([ursula](https://github.com/blueboxgroup/ursula)) to support RedHat Enterprise Linux as well as Ubuntu ([see](https://www.ibm.com/blogs/bluemix/2017/04/ibm-bluemix-private-cloud-red-hat/)), with full STIG compliance and Chef Inspec for auditing.

* Recognized the need for a unified SRE Operations Platform to support growth and built SiteController ([cuttle](https://github.com/IBM/cuttle)) and architected and built it, later forming and leading a team to maintain and develop it further.

* Successfully led the effort to make Blue Box OpenStack installable in customer data centers with no Internet access utilizing SiteController and overhauling large parts of Ursula.

### Rackspace

* Built the initial Build/Push/Run workflow for the [now mostly defunct] OpenStack PaaS project (Solum) utilizing Docker and the Cedarish style workflow demonstrated by Dokku and DEIS.

* Worked on the OpenStack nova-docker driver, and was the first [that I know of] person to successfully run [OpenStack in Docker](https://github.com/paulczar/dockenstack) containers.

* Maintained Community Chef Cookbooks for Elasticsearch, Logstash, and Kibana.

### Electronic Arts

* Planned and Executed migration of live game services for multiple games from their existing expensive datacenters to spare capacity from the cheaper BioWare datacenters.

* Built and deployed dozens of websites for game code redemptions and blogs into Amazon using Rightscale.

* Helped build and scale Sim City Online servers in Amazon using Rightscale, implemented monitoring and logging systems to help debug and discover load and performance issues during launch instability.

### BioWare – Star Wars The Old Republic (SWTOR)

* Supported the studio during development.   Lead the Online Operations team through purchasing and deploying over 6,000 servers in four data centers to run the online environment. Ensured a successful and glitch free  launch of SWTOR on the 20th December 2011.

* Built a large cluster of Xen Hypervisors to provide virtual game servers for development and wrote scripts for deploying game databases from SAN snapshots ( reducing storage requirements from 36Tb to less than 1Tb ).

* Successfully deployed a proof of concept private cloud with CloudStack to further increase our Virtualization abilities and create a self-service portal for our developers.

## Employment History

### Red Hat - April 2021 to Current

*Austin, Texas*

Principal Managed OpenShift Black Belt

### VMWare / Pivotal Software - Nov 2017 to April 2021

*Austin, Texas*

Staff Technologist / Developer Advocate

### BlueBox an IBM Company – Nov 2014 to Nov 2017

*Austin, Texas*

Technical Lead, IBM Cloud Developer Labs,

Architect / Senior DevOps Engineer,

Technical lead of Site Controller team.

### Rackspace – Nov 2013 to Nov 2014

*Austin, Texas*

Senior Operations Engineer

### EA / BioWare – April 2008 to Nov 2013

*Austin, Texas*

Manager, Systems Engineering *( March 2012 to current )*

Lead Systems Engineer  *( 2010  to March 2012 )*

Senior Systems Engineer  ( 2008 to 2010 )

### Older

2004 - 2008 : IT Manager, Pandemic Studios.
2001 - 2004 : NOC Manager iTEL Community Telco
2000 – 2001 : Systems Administrator / Web Application Developer BMC Networks
1999 – 2000 : Systems Administrator, Global Info-Links
1998 – 1999 : Computer Technician, Altech Computers
1998             : Computer Technician, Harvey Norman
1997 - 1998  : OzNetCom

### References

Available on request.
