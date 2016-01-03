---
date: "2016-01-02T14:44:30-06:00"
title: "Deploying a HA Docker Swarm Cluster"
categories: [ "openstack" ]
draft: true
---

Given Docker's propensity for creating easy to use tools it shouldn't come as a surprise that Docker Swarm is one of the easier to understand and run of the "Docker Clustering" options currently out there. I recently built some [Terraform](http://terraform.io) configs for deploying a [Highly Available Docker Swarm cluster on Openstack](https://github.com/openstack/osops-tools-contrib/tree/master/terraform/dockerswarm-coreos) and learned a fair bit about Swarm in the process.

This guide is meant to be a platform agnostic howto on installing and running a Highly Available Docker Swarm to show you the ideas and concepts that may not be as easy to understand from just reading some config management code.

## CoreOS

The reason for using [CoreOS](http://coreos.com) here is that to make Swarm run in High Availability mode as well as being able to support docker networking between hosts we need to use service discovery.  We can choose to use `etcd`, `consul`, or `zookeeper` here, CoreOS comes with `etcd` thus makes it an excellent choice for running Docker Swarm.

You will need three servers capable of running [CoreOS](http://coreos.com).  See the "Try Out CoreOS" section of their website for various installation methods for different infrastructure. For this guide I will use the official [CoreOS Vagrant Example](https://github.com/coreos/coreos-vagrant).

_skip the rest of this section if you install CoreOS for a different platform_

Clone down the Vagrant example:

```
$ git clone https://github.com/coreos/coreos-vagrant.git vagrant-docker-swarm 
Cloning into 'vagrant-docker-swarm'...
remote: Counting objects: 411, done.
remote: Total 411 (delta 0), reused 0 (delta 0), pack-reused 411
Receiving objects: 100% (411/411), 100.33 KiB | 0 bytes/s, done.
Resolving deltas: 100% (181/181), done.
Checking connectivity... done.
cd vagrant-docker-swarm
```

Edit the `Vagrantfile` to set `$num_instances = 3`:

_on Unix-like systems you can do this easily with sed_

```
sed -i 's/\$num_instances = 1/\$num_instances = 3/' Vagrantfile
```

Get a new etcd discovery-url:

_if you are on a windows box and don't have curl you can paste the url into a web browser to get the discovery-url_

```
$ curl https://discovery.etcd.io/new\?size\=3
https://discovery.etcd.io/6a9c62105f04dac40a29b90fbed322ef
```

Create a cloud-init file called `user-data` in the base of the repo using the discovery-url from above:

```
#cloud-config

coreos:
  etcd2:
    discovery: https://discovery.etcd.io/888fd1e440faf680a7abb3fd934da6fd
    advertise-client-urls: http://$public_ipv4:2379
    initial-advertise-peer-urls: http://$public_ipv4:2380
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    listen-peer-urls: http://$public_ipv4:2380,http://$public_ipv4:7001
  units:
    - name: etcd2.service
      command: start

```

Start up the CoreOS VMs and log into the first one to check everything worked ok:

```
$ vagrant up
Bringing machine 'core-01' up with 'virtualbox' provider...
Bringing machine 'core-02' up with 'virtualbox' provider...
Bringing machine 'core-03' up with 'virtualbox' provider...
...
$ vagrant ssh core-01
$ etcdctl member list
3c5901a3db54efa3: name=f1bae7bba7714ed7b4585c6b1256ddb2 peerURLs=http://172.17.8.101:2380 clientURLs=http://172.17.8.101:2379
9eeb141350af8439: name=5c8e57890d114d7d9d7aef662033a6e0 peerURLs=http://172.17.8.103:2380 clientURLs=http://172.17.8.103:2379
ebcc652087dfe6e8: name=de426249d3b34e23a5706d99b4900665 peerURLs=http://172.17.8.102:2380 clientURLs=http://172.17.8.102:2379
```

## Docker Swarm

Now that we have several CoreOS servers with a working etcd cluster we can move on to setting up Docker Swarm.

We need to modify docker to listen on tcp port `2376` as well as registering itself to service discovery (which will allow us to set up overlay networking later on).  We do this by creating a file `custom.conf` in `/etc/systemd/system/docker.service.d/` on each server.

_if not using vagrant change `eth1` to match the primary interface for your server_

```
[Service]
Environment="DOCKER_OPTS=-H=0.0.0.0:2376 -H unix:///var/run/docker.sock --cluster-advertise eth1:2376 --cluster-store etcd://127.0.0.1:2379"
```

We then need to reload the `systemctl` daemon and then restart docker for these changes to take effect.

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Check that you can access docker via tcp on one of your hosts:

```
$ docker -H tcp://172.17.8.101:2376 info
Containers: 0
Images: 0
Engine Version: 1.9.1
Storage Driver: overlay
 Backing Filesystem: extfs
Execution Driver: native-0.2
Logging Driver: json-file
Kernel Version: 4.3.3-coreos
Operating System: CoreOS 899.1.0
CPUs: 1
Total Memory: 997.4 MiB
Name: core-01
ID: BK64:WF3J:5JU6:VYLI:YJSO:CAQH:HPYM:MPTG:FMTA:VLE3:HSMP:F4VQ
Cluster store: etcd://127.0.0.1:2379/docker

```

We're now ready to run Docker Swarm itself. There are two extra components to running Docker Swarm, a Swarm Agent and a Swarm Manager.

The Swarm Agent watches the local Docker service via it's TCP port and registers it into service discovery (etcd in our case).  We will run this on each server like so:

_set the --addr= argument to match the primary IP of each node_

```
$ docker run -d --name swarm-agent \
    --net=host swarm:latest \
        join --addr=172.17.8.101:2376 \
        etcd://127.0.0.1:2379
```

The Swarm Manager watches service discovery and exposes a TCP port (2375) which when accessed by a Docker client will perform actions and schedule containers across the Swarm cluster.

To ensure High Availability of our cluster we'll run a Swarm Manager on each server:

```
$ docker run -d --name swarm-manager 
    --net=host swarm:latest manage \
    etcd://127.0.0.1:2379
```

Assuming everything went smoothly we can now access the swarm cluster via the Swarm Managers TCP port on any of the servers:

```
$ docker -H tcp://172.17.8.101:2375 info
Containers: 6
Images: 5
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 3
 core-01: 172.17.8.101:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.023 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.3.3-coreos, operatingsystem=CoreOS 899.1.0, storagedriver=overlay
 core-02: 172.17.8.102:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.023 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.3.3-coreos, operatingsystem=CoreOS 899.1.0, storagedriver=overlay
 core-03: 172.17.8.103:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.023 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.3.3-coreos, operatingsystem=CoreOS 899.1.0, storagedriver=overlay
CPUs: 3
Total Memory: 3.068 GiB
Name: core-01
```

Our next step is to create an overlay network using the `docker network` command:

```
$ docker -H tcp://172.17.8.101:2375 network create --driver overlay my-net
614913b275dee43a63b48d08b4f5e52f7c0e531d70c63eeb8bb35624470da0c4

$ docker -H tcp://172.17.8.101:2375 network ls                            
NETWORK ID          NAME                DRIVER
86ecb0cf32c6        core-02/none        null                
c7a291ed8366        core-01/host        host                
3747364c5961        core-03/none        null                
8245d6d3ac67        core-02/host        host                
614913b275de        my-net              overlay             
61ead145e9dd        core-01/bridge      bridge              
c9457c4f4588        core-03/bridge      bridge              
b8a6c75cb3b9        core-03/host        host                
bdc4d5ccd778        core-02/bridge      bridge              
66afdc892361        core-01/none        null
```

Finally we'll create a Container on one host and then check that it is accessible from another:

_replace the node==XXXX argument with the hostname of one of your hosts, make sure to use a different node for each docker command_

```
$ docker run -it --name=web --net=my-net \
    -H tcp://172.17.8.101:2375 \
    --env="constraint:node==core-01" nginx
e0fe18c946a5692806608f939d4d6f31c670e3f42bf3942a77142bed2095983e

$ docker run -it --rm --net=my-net \
    -H tcp://172.17.8.101:2375 \
    --env="constraint:node==core02" busybox wget -O- http://web
Connecting to web (10.0.0.2:80)
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

If you've been following along you have successfully deployed a Highly Available Docker Swarm cluster.  From here you could use a load balancer to load balance the Swarm Manager port (2375) or even use Round Robin DNS.

You may have notice there is no authentication or authorization on this and anybody with a Docker binary and TCP access to your hosts could spin up docker containers. This is fairly easily fixed by using Docker's TLS cert based authorization, I'll cover setting that up in a future blog post.
