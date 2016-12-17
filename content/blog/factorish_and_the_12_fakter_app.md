+++
date = "2015-01-06T13:29:27-06:00"
title = "Factorish and The Twelve-Fakter App"
+++

Unless you've been living under a rock (in which case I envy you) you've heard a fair bit about The [Twelve-Factor App](http://12factor.net). A wonderful stateless application that is completely disposable and can run anywhere from your own physical servers to [Deis](http://deis.io), [Cloud Foundry](http://cloudfoundry.org) or [Heroku](http://heroku.com).

Chances are you're stuck writing and running an application that is decidely not 12Factor, nor will it ever be.  In a perfect world you'd scrap it and rewrite it as a dozen microservices that are loosely coupled but run and work indepently of eachother. The reality however is you could never get the okay to do that.

<!--more-->

Fortunately with the rise of [Docker](http://docker.com) and its ecosystem it has become easier to not only write 12Factor apps, but also to fake it by producing a Docker container that acts like a 12Factor app, but contains something that is decidedly not.  I call this the 12Fakter app.

I've been playing with this concept for a while, but over Christmas I spent a bunch of time trying to figure out the best ways to fake out the 12 Factors and feel that I've come up with something that works pretty well and in the process created a Vagrant based development sandbox called [Factorish](http://github.com/paulczar/factorish) which I used to create [12fakter-wordpress](http://github.com/paulczar/12fakter-wordpress) and [elk_confd](https://github.com/paulczar/docker-elk_confd).


## Fakter I. Codebase

__One codebase tracked in revision control, many deploys__

The goal here is to have both your app and deployment tooling in the same codebase which is stored in source control.  This means adding a `Dockerfile`, and `Vagrantfile` and other pieces of tooling into your codebase.  If however you have a monolithic codebase that contains more than just your app you can create a seperate codebase ( use git! ) containing this tooling and have that tooling collect the application from its existing codebase.

You should be able to achieve this by either merging [Factorish](http://github.com/paulczar/factorish) into your existing git repo,  or fork it and use the `Dockerfile` in it to pull the actual application code in as part of the build process.

## Fakter II. Dependencies

__Explicitly declare and isolate dependencies__

This is a really easy win with Docker,  The very nature of Docker both Explicitly declares your dependencies in the form of the `Dockerfile` and Isolates them in the form of the built Docker image.

### Declaration

#### /app/example/Dockerfile
```
FROM python:2

# Base deps layer
RUN \
  apt-get update && apt-get install -yq \
  make \
  ca-certificates \
  net-tools \
  sudo \
  wget \
  vim \
  strace \
  lsof \
  netcat \
  lsb-release \
  locales \
  socat \
  supervisor \
  --no-install-recommends && \
  locale-gen en_US.UTF-8

# etcdctl and confd layer
RUN \
  curl -sSL -o /usr/local/bin/etcdctl https://s3-us-west-2.amazonaws.com/opdemand/etcdctl-v0.4.6 \
  && chmod +x /usr/local/bin/etcdctl \
  && curl -sSL -o /usr/local/bin/confd https://github.com/kelseyhightower/confd/releases/download/v0.7.1/confd-0.7.1-linux-amd64 \
  && chmod +x /usr/local/bin/confd

ADD . /app
WORKDIR /app

# app layer
RUN \
  useradd -d /app -c 'application' -s '/bin/false' app && \
  chmod +x /app/bin/* && \
  pip install -r /app/example/requirements.txt

# Define default command.
CMD ["/app/bin/boot"]

# Expose ports.
EXPOSE 8080
```

You might notice I have sets of commands joined together with `&&` in my `Dockerfile`, I do this to control the docker layers more to try and end up with fewer more meaningful layers.

### Isolation

```
$ docker build -t factorish/example example
Sending build context to Docker daemon 20.99 kB
Sending build context to Docker daemon
Step 0 : FROM python:2
 ---> 96e13ecb4dba
...
...
Step 8 : EXPOSE 8080
 ---> Running in 8dc9a04eaf78
 ---> 374cb835239c
Removing intermediate container 8dc9a04eaf78
Successfully built 374cb835239c
```

## Fakter III. Configuration

__Store config in the environment__

Another easy win with Docker.   You can pass in environment variables in the `Dockerfile` as we as when running the docker container using the `-e` option like this:

```
$ docker run -d -e TEXT=bacon factorish/example
```

However chances are your app reads from a config file rather than environment variables. There are [at least] two fairly simple ways to achieve this.

### sed inline replacement

use a startup script to edit your config file and replace values in it with the values of the environment variables using `sed` before runnin your app:

#### /app/bin/boot
```
#!/bin/bash
sed -i "s/xxxTEXTxxx/${TEXT}" /app/example/example.conf
python /app/example/app.py
```

### confd templating

[confd](https://github.com/kelseyhightower/confd) is a tool written specifically for templating config files from data sources such as environment variables.  This is a much better option as it also opens up the ability to use service discovery tooling like [etcd](https://coreos.com/using-coreos/etcd/) (also supported in Factorish) rather than environment variables.

#### /app/conf.d/example.conf.toml
```
[template]
src   = "example.conf"
dest  = "/app/example/example.conf"
keys = ["/services/example"]
```

#### /app/templates/example.conf
```
[example]
text: {{ getv "/services/example/text" }}
```

_The `{{ }}` syntax above is the golang/confd macros used to perform tasks like fetching variables from etcd or environment._

#### /app/bin/boot
```
#!/bin/bash
confd -onetime
python /app/example/app.py
```

## Fakter IV. Backing Services
_Treat backing services as attached resources_

Anything that is needed to store persistent data should be treated as an external dependency to your application.  As far as your app is concerned there should be no difference between a local MySQL server or Amazon's RDS.

This is easier for some backing services than others.  For example if your app requires a MySQL database its relatively straight forward.  Whereas a local filesystem for storing images is harder, but can be solved:

* Docker: volume mounts, data containers
* Remote Storage: netapp, nfs, fuse-s3fs
* Clustered FS: drdb, gluster
* Ghetto: rsync + concerned

The docker volume mounts actually work really well in a vagrant based development environment because you can pass your code all the way into the container from your workstation,  however there are definitely some security considerations to think about if you want to do volume mounts in production.

### Example

A fictional __PHP__ based blog about bacon requires a database and a filestore:

#### /app/templates/config.php
```
define('DB_NAME', '{{ getv "/db/name" }}');
define('DB_USER', '{{ getv "/db/user" }}');
define('DB_PASSWORD', '{{ getv "/db/pass" }}');
define('DB_HOST', '{{ getv "/db/host" }}');
```

#### Docker Run command
```
$ docker run -d -e DB_NAME=bacon -e DB_USER=bacon \
  -e DB_PASSWORD=bacon $DB_HOST=my.database.com \
  -v /mnt/nfs/bacon:/app/bacon factorish/bacon-blog
```

confd will use the environment variables passed in via the `docker run` command to fill out the variables called in the `{{ }}` macros.  Note that confd transforms the environment variables so that the environment variable `DB_USER` will be read by `{{ getv "/db/user" }}`.  This is done to normalize the macro across the various data source options.

## Fakter V. Build, Release, Run

__Strictly separate build and run stages__

### Build

Converts a code repo into an executable bundle. Sound familiar?  Yup, we've already solved this with our `Dockerfile`.

### Release

Takes the build and combines it with the current configuration. In a purely docker based system this can be split between the __Build__ (versioning and defaults) and __Run__ (current config) stages. However systems like Heroku and Deis have a seperate step for this which they handle internally.

### Run

Runs the application by launching a set of the app's processes against a selected release.  In a docker based system this is simply the `$ docker run` command which can be called via a deploy script, or a init script (systemd/runit) or a scheduler like [fleet](https://coreos.com/using-coreos/clustering/) or [mesos](http://mesos.apache.org/).

## Fakter VI. Processes

__Execute the app as one or more stateless processes__

Your application inside the docker container should behave like a standard linux process running in the foreground and be stateless and share-nothing.  Being inside a docker container means that this is hidden and therefore we can fairly easily fake this but you do need to think about process management and logging which are discussed later and is further explored [here](http://tech.paulcz.net/2014/12/multi-process-docker-images-done-right/).

## Fakter VII. Port binding
__Export services via port binding__

Your application should appear to be completely self contained and not require runtime injection of a webserver.  Thankfully this is pretty easy to fake in a docker container as any extra processes are isolated in the container and effectively invisible to the outside.

It is still preferable to use a native language based web library such as jetty (java) or flask (python) but for languages like PHP using apache or nginx is ok.

Docker itself takes care of the port binding by use of the `-p` option on the command line.  It's useful to register the port and host IP to somewhere ( etcd ) to allow for loadbalancers and other services to easily locate your application.

## Fakter VIII. Concurrency
__Scale out via the process model__

We should be able to scale up or down simply by creating or destroying docker containers containing the application.  Any upstream load balancers as an external dependency would need to be notified of the container starting ( usually a fairly easy API call) and stopping.  But these are external dependencies and should be solved outside of your application itself.

Inside the container your application should not daemonize or write pid files (if unavoidable, not too difficult to script around) and use tooling like `upstart` or `supervisord` if there is more than one process that needs to be run.

## Fakter IX. Disposability
__Maximize robustness with fast startup and graceful shutdown__

Docker helps a lot with this.   We want to ensure that we're optimized for fast yet reliable startup as well as graceful shutdown.  Your app should be able to be shut down gracefully when `docker kill` is called and just as importantly there should be minimal if any external effect if the application crashes or stops ungracefully.

The container itself should kill itself if the app inside it stops working right.  If your app is running behind a [supervisor](http://tech.paulcz.net/2014/12/multi-process-docker-images-done-right/) this can be a achieved with a really lightweight healthcheck script like this.

#### /app/bin/healhthcheck
```
#!/bin/bash
while [[ ! -z $(netstat -lnt | awk "\$6 == \"LISTEN\" && \$4 ~ \".$PORT\" && \$1 ~ \"tcp.?\"") ]] ; do
  [[ -n $ETCD_HOST ]] && etcdctl set /service/web/hosts/$HOST $PORT --ttl 10 >/dev/null
  sleep 5
done
kill `cat /var/run/supervisord.pid`
```

You'll note that I'm also publishing host and port values to etcd if `$ETCD_HOST` is set.  This can then be used to notify loadbalancers and the like when services start or stop.

## Fakter X. Dev/prod parity

__Keep development, staging, and production as similar as possible__

By following the previous fackters we've done most of the work to make this possible.  We use Vagrant in development to deploy your app (and any backing services) using the appropriate provisioning methodology ( the same ones we'd use for production).

By wrapping the application in a docker container it is portable across just about any system that is capable of running docker.

By provisioning with the same tooling to both dev and prod (and any other envs),  any deployment of development (should happen frequently) is also a test of most of the tooling used to deploy to production.

## Fakter XI. Logs
__Treat logs as event streams__

Your application ( even inside the container ) should always log to stdout. By writing to stdout of your process we can utilize the docker logging subsystem which when combined with tooling like [logspout](https://registry.hub.docker.com/u/progrium/logspout/) makes it very easy to push all logs to a central system.

If your app _has_ to write to a logfile you should be able to configure that log file to be `/dev/stdout` which should cause it to write to stdout of the process. If your app only writes to syslog then configure it to write to a remote syslog. Basically do whatever you can to ensure you don't log to the local filesystem.

### Example

This example shows running `Supervisord` as your primary process in the docker container and `nginx` writing logs to stdout which in turn are written to the containers `stdout`.  A more thorough writeup on using [supervisor](http://tech.paulcz.net/2014/12/multi-process-docker-images-done-right/) inside docker containers can be found [here](http://tech.paulcz.net/2014/12/multi-process-docker-images-done-right/):

#### /etc/supervisor/conf.d/nginx
```
[supervisord]
logfile=/dev/null
pidfile=/var/run/supervisord.pid
nodaemon=true

[program:nginx]
command=/usr/sbin/nginx
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
auto_start=true
autorestart=true
user=root
```

#### /etc/nginx/sites-enabled/app
```
worker_processes 1;
daemon off;
error_log /dev/stdout;
http {
  access_log /dev/stdout;
  server {
    listen            *:8080;
    root              /app/bacon-blog;
    index             index.php;
  }
}
```

For a more detailed post on using logspout to produce consumable logs check out [@behemphi](https://twitter.com/behemphi)'s blog post - [Docker Logs – Aggregating with Ease](http://stackengine.com/docker-logs-aggregating-ease/)


## Fakter XII. Admin processes

__Run admin/management tasks as one-off processes__

This one is pretty easy.  Tasks such as database migrates should be run in one off throw-away containers.

```
$ docker run -t -e DB_SERVER=user@pass:db.server.com myapp:1.3.2 rake db:migrate
```

## Conclusion

Most of the fakters above are relatively straight forward to utilize and can be built upon slowly, no need to perfect things before working on them.  They can also be utilized with any existing provisioning / config management tooling that you already have.

If you're already using [chef](http://chef.io) for deploying your application you can use the [docker cookbook](https://supermarket.chef.io/cookbooks/docker) to start running docker containers instead and write out confd templates rather than the final config file which confd will then use to do the final configuration of your app from the environment variables you pass through to the `docker_run` resource in the cookbook.

Making your application act like a 12Factor app may not be enough to run it on a purely hosted PAAS like Heroku, but chances are you'll be able to run it on a Docker based PAAS like Deis.  You can go full stack with Mesos or CoreOS+Fleet+ETCD or you can stick to Ubuntu servers running docker.

The flexibility that the 12fakter application gives you means that you can move to a more modern infrastructure at your own pace when it makes sense without having to abandon or completely rewrite your existing applications.

Please check out [Factorish](http://github.com/paulczar/factorish) and some of the example 12fakter apps like [12fakter-wordpress](http://github.com/paulczar/12fakter-wordpress) and [elk_confd](https://github.com/paulczar/docker-elk_confd). to see how easy it can be to start making your applications act like 12Factor apps.