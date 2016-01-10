---
date: "2016-01-10T10:22:22-06:00"
title: "Flexible Private Docker Registry Infrastructure"
categories: [ "docker", "registry" ]
---

Previously I showed how to run a [basic secure Docker Registry](http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/).  I am now going to expand on this to show you something that you might use in production as part of your CI/CD infrastructure.

The beauty of running Docker is that you _can_ push an image from a developer's laptop all the way into production which helps ensure that what you see in development and your various test/qa/stage environments are exactly the same as what you run in production.

So they tell you anyway. The reality is that you don't ever want to push an image built on a developer's machine into production as you can't be sure what is in it.  Instead you want to have a trusted build server build images from a `Dockerfile` in your git repository and have it promoted through your environments from there.

To ensure the integrity of your images you'll want to run a Docker Registry that can be reached by all of your servers (and potentially people), but can only be written to by your build server (and/or an administrative user).

You could run your [Docker Registry](https://www.docker.com/docker-registry) behind a [complicated reverse proxy](https://docs.docker.com/registry/recipes/) and create rules about who can GET/POST/etc through to the [Docker Registry](https://www.docker.com/docker-registry) however we can use the magic of "[The Cloud](https://github.com/panicsteve/cloud-to-butt)" to reduce the complexity and thus the need for a reverse proxy.

You will want to use either the [Openstack Swift](https://wiki.openstack.org/wiki/Swift) or the [Amazon S3](https://aws.amazon.com/s3/) object storage driver for the [Docker Registry](https://www.docker.com/docker-registry). I will demonstrate using Swift, but using S3 should be very similar.

You will of course want to also build all of these servers with Configuration Management including the commands to actually run the [Docker Registry](https://www.docker.com/docker-registry).

## Build Server(s)

For your build server(s) you'll want to be running an OS with Docker installed on it. I use the [Jenkins](https://hub.docker.com/_/jenkins/) Docker image on [CoreOS](http://coreos.com/) for both my Jenkins Master and Slaves, however this is just personal preference.

On each server you want to run a [Docker Registry](https://www.docker.com/docker-registry) with your Swift credentials passed through to it. Since we're only accessing this via `127.0.0.1` we do not need to secure it with TLS or authentication.

Run the following on each build server to run the Registry backed by Swift, replacing the OpenStack credentials with your own:

```
build01$ docker run -d \
              -p 127.0.0.1:5000:5000 \
              --name registry \
              --restart always \
              -e REGISTRY_STORAGE=swift \
              -e REGISTRY_STORAGE_SWIFT_USERNAME=${OS_USERNAME} \
              -e REGISTRY_STORAGE_SWIFT_PASSWORD=${OS_PASSWORD} \
              -e REGISTRY_STORAGE_SWIFT_TENANT=${OS_TENANT} \
              -e REGISTRY_STORAGE_SWIFT_AUTHURL=${OS_AUTH_URL} \
              -e REGISTRY_STORAGE_SWIFT_CONTAINER=docker-registry \
              registry:2
```

Push an image to make sure it worked:

```
build01$ docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
Digest: sha256:78a756d480bcbc35db6dcc05b08228a39b32c2b2c7e02336a2dcaa196547a41d
Status: Downloaded newer image for alpine:latest
$ docker tag alpine 127.0.0.1:5000/alpine
$ docker push 127.0.0.1:5000/alpine
The push refers to a repository [127.0.0.1:5000/alpine] (len: 1)
74e49af2062e: Pushed 
latest: digest: sha256:a96155be113bb2b4b82ebbc11cf1b511726c5b41617a70e0772f8180afc72fa5 size: 1369
```

If you have more that one build server try to pull the image from one of the others, since we're backing the [Docker Registry](https://www.docker.com/docker-registry) with an object store they should retrieve it just fine:

```
build02$ docker pull 127.0.0.1:5000/alpine
Using default tag: latest
latest: Pulling from alpine

340b2f9a2643: Already exists 
Digest: sha256:a96155be113bb2b4b82ebbc11cf1b511726c5b41617a70e0772f8180afc72fa5
Status: Downloaded newer image for 127.0.0.1:5000/alpine:latest
```

## Regular Server(s)

We have a couple of options here.  You can run a [Docker Registry](https://www.docker.com/docker-registry) on each server listening only on localhost, or you can run one or more of them on their own servers that will listen on an IP and be secured with TLS.

We'll cover the former use case, for the latter use case you can adapt the instructions found [at my previous blog post](http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/).

The important step in either case is to start the Registry as read-only so that regular servers cannot alter the contents of the Registry.

The [Docker Registry](https://www.docker.com/docker-registry) is fairly light-weight when the files are in external storage and thus will use a neglible amount of your system resources and provides the advantages and security of running the registry on localhost and not needed to set `--insecure-registry` settings or worrying about TLS certs for the docker daemon.

```
$ docker run -d \
      -p 127.0.0.1:5000:5000 \
      --name registry \
      --restart always \
      -e REGISTRY_STORAGE_MAINTENANCE_READONLY='enabled: true' \
      -e REGISTRY_STORAGE=swift \
      -e REGISTRY_STORAGE_SWIFT_USERNAME=${OS_USERNAME} \
      -e REGISTRY_STORAGE_SWIFT_PASSWORD=${OS_PASSWORD} \
      -e REGISTRY_STORAGE_SWIFT_TENANT=${OS_TENANT} \
      -e REGISTRY_STORAGE_SWIFT_AUTHURL=${OS_AUTH_URL} \
      -e REGISTRY_STORAGE_SWIFT_CONTAINER=docker-registry \
      registry:2
```

With `REGISTRY_STORAGE_MAINTENANCE_READONLY='enabled: true` set, when we try to push to the registry it should fail:

```
$ docker push 127.0.0.1:5000/alpine
The push refers to a repository [127.0.0.1:5000/alpine] (len: 1)
f4fddc471ec2: Preparing 
Error parsing HTTP response: invalid character 'M' looking for beginning of value: "Method not allowed\n"
```

## User Access to Registry:

If you want to provide access to regular users and don't mind maintaining the password files locally you can adapt my [basic secure Docker Registry](http://tech.paulcz.net/2016/01/deploying-a-secure-docker-registry/) blog post to use the object storage backend.

Assuming you've followed the instructions provided to create the TLS certificates you can run two [Docker Registry](https://www.docker.com/docker-registry)s each pointing at a different `htpasswd` file.

These can run on the same server, or on seperate servers.  They can also be run on multiple servers that are load balanced via an external load balancer or via round-robin-dns for high availability.

### Read only Users      

```
$ docker run -d \
      -p 443:5000 \
      --name registry \
      --restart always \
      -v /opt/registry \
      -e REGISTRY_STORAGE_MAINTENANCE_READONLY='enabled: true' \
      -e REGISTRY_STORAGE=swift \
      -e REGISTRY_STORAGE_SWIFT_USERNAME=${OS_USERNAME} \
      -e REGISTRY_STORAGE_SWIFT_PASSWORD=${OS_PASSWORD} \
      -e REGISTRY_STORAGE_SWIFT_TENANT=${OS_TENANT} \
      -e REGISTRY_STORAGE_SWIFT_AUTHURL=${OS_AUTH_URL} \
      -e REGISTRY_STORAGE_SWIFT_CONTAINER=docker-registry \
      -e REGISTRY_AUTH=htpasswd \
      -e "REGISTRY_AUTH_HTPASSWD_REALM=Admin Registry Realm" \
      -e REGISTRY_AUTH_HTPASSWD_PATH=/opt/registry/auth/admin.htpasswd \
      -e REGISTRY_HTTP_SECRET=qerldsljckjqr \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/opt/registry/ssl/cert.pem \
      -e REGISTRY_HTTP_TLS_KEY=/opt/registry/ssl/key.pem \
      registry:2
```

### Admin Read/Write

```
$ docker run -d \
      -p 444:5000 \
      --name registry \
      --restart always \
      -v /opt/registry \
      -e REGISTRY_STORAGE=swift \
      -e REGISTRY_STORAGE_SWIFT_USERNAME=${OS_USERNAME} \
      -e REGISTRY_STORAGE_SWIFT_PASSWORD=${OS_PASSWORD} \
      -e REGISTRY_STORAGE_SWIFT_TENANT=${OS_TENANT} \      
      -e REGISTRY_STORAGE_SWIFT_AUTHURL=${OS_AUTH_URL} \
      -e REGISTRY_STORAGE_SWIFT_CONTAINER=docker-registry \
      -e REGISTRY_AUTH=htpasswd \
      -e "REGISTRY_AUTH_HTPASSWD_REALM=Read Only Registry Realm" \
      -e REGISTRY_AUTH_HTPASSWD_PATH=/opt/registry/auth/users.htpasswd \
      -e REGISTRY_HTTP_SECRET=hlyrehbrvgszd \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/opt/registry/ssl/cert.pem \
      -e REGISTRY_HTTP_TLS_KEY=/opt/registry/ssl/key.pem \
      registry:2
```

Before pushing or pull images to these registries you'll need to log in using `docker login myregistrydomain.com:443` or `docker login myregistrydomain.com:444`.

By using external storage for the Registry we have increased our ability to run a resiliant Docker Registry with no single points of failure. All of the servers access the registry itself via localhost which means they have almost no reliance on external systems (except for a very robust object storage platform) and no need for complicated authentication systems.

We also provide access to both Admin (read/write) and Regular (read-only) users via `htpasswd` files and `TLS` certificates/encryption which can be managed by Configuration Management.

It goes without saying that you should further lock down all of these services with network based access restrictions in the form of Firewall/IPTables/Security-Groups so that only certain trusted networks can access any of the public endpoints we have created.
