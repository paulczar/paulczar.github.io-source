---
title: "EZBake - A new way to converge docker containers with chef"
date: "2014-05-13"
categories: [chef, devops, docker] 
slug: ezbake-a-new-way-to-converge-docker-containers-with-chef
---


`EZ Bake` came from an idea I had while watching the [HangOps](https://twitter.com/hangops) [episode 2014-04-11](https://www.youtube.com/watch?v=clLFKIeSADo&feature=youtu.be) in which they were talking about `Docker` and Config Management being complementary rather than adversary.

I have expermented with using `Chef` and `Docker` together in the [past](/2013/09/creating-immutable-servers-with-chef-and-docker-dot-io.html) but wanted to tackle the problem from a slightly different angle.  I've recently been working on some PAAS stuff, both [Deis](http://deis.io) and [Solum](http://solum.io) these both utilize the tooling from [Flynn](https://github.com/flynn/flynn) which builds heroku style `buildpacks` in `Docker`.

<!--more-->

EZ Bake takes chef recipes designed for `chef-solo` ( but could easily be extended to do the same for `chef-zero`, or `chef-client` with a server) in a tarball via `stdin` and converges a docker node using that recipe.

This methodology seems a little weird at first,  but it gives you the ability to ship your Chef cookbooks as self-contained tarballs, or even more interestingly use the `git archive` command from your git repository to do this automatically and then pipe that directly to the `docker run` command.

In order to recognize and run your cookbook ( or repo ) it needs to contain the following files: `Berksfile`, `solo.json`, `solo.rb` in the root of your cookbook.   There is some provision for providing different locations for these via environment variables.   This is pre-ChefDK and will probably become easier with ChefDK.

I have provided an example in the ezbake repo that will install Java7 in the container.  

This example shows:

*  Converging a container using a local chef recipe
*  Committing the container to an image on completion
*  Removing the build container
*  Running the new image

```
$ git clone paulczar/ezbake
$ cd ezbake/examples
$ ID=$(tar cf - . | sudo docker run -i -a stdin paulczar/ezbake) \
  && sudo docker attach $ID \
  && sudo docker commit $ID java7 
  && sudo docker rm $ID

Running Berkshelf to collect your cookbooks:
Installing java (1.22.0) from site: 'http://cookbooks.opscode.com/api/v1/cookbooks'
Converging your container:
[2014-04-12T22:10:24+00:00] INFO: Forking chef instance to converge...
....
[2014-04-12T22:16:52+00:00] INFO: Chef Run complete in 154.563192281 seconds
[2014-04-12T22:16:52+00:00] INFO: Running report handlers
[2014-04-12T22:16:52+00:00] INFO: Report handlers complete

$ sudo docker run -t java7 java -version
java version "1.7.0_51"
Java(TM) SE Runtime Environment (build 1.7.0_51-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode)

```

This could easily be built into a CI pipeline.   a git webhook could call jenkins which would clone the repo and then use a command like  `git archive master | docker run -i -a stdin paulczar/ezbake` to converge a container from it.  

It could also very easily be used in `Deis` or `Solum` as an alternative to a Heroku buildpack.
