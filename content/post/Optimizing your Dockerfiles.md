+++
date = "2015-03-07T13:25:29-06:00"
title = "Optimizing your Dockerfiles"
+++

Docker images are "supposed" to be small and fast. However unless you're precompiling GO binaries and dropping them in the `busybox` image they can get quite large and complicated. Without a well constructed `Dockerfile` to improve build cache hits your docker builds can become unnecessarily slow.

`Dockerfile`'s are regularly [and incorrectly] treated like `bash` scripts and therefore are often written out as a series of commands which you would `curl | sudo bash` from a website to install.  This usually makes for an inefficient and slow `Dockerfile`

<!--more -->
## Order Matters

When you're building a new `Dockerfile` for an application there can be a lot of trial and error in determining what packages are needed and what commands need to run. Optimizing your `Dockerfile` ensures that the build cache will hit more often and each build between changes will be faster.  

The general rule of thumb is to sort your commands by frequency of change, the time it takes to run the command and how sharable it is with other images.

This means that commands like `WORKDIR`, `CMD`, `ENV` should go towards the bottom while a `RUN apt-get -y update` should go towards the top as it takes longer to run and can be shared with all of your images.

Finally any `ADD` ( or other commands that invalidate cache ) commands should go as far down the bottom as possible as this is where you're likely to make lots of changes that will invalidate the cache of subsequent commands.

## Choose your base image wisely

There's a lot of base images to choose from from the bare OS images like `ubuntu:trusty` to application specific ones for `python:2` or `java:7`.  Common sense might tell you to use `ruby:2` to run an ruby based app and `python:3` to run a python app.  However now you have two base images with little in common that you need to download and build.  Instead if you use `ubuntu:trusty` for both then you only need to download the base image once.

## Use Layers to your advantage

Each command in a `Dockerfile` is an extra layer. You can very quickly end up with an image that's 30+ layers.  This is not necessarily a problem, but by joining `RUN` commands together, and using a single `EXPOSE` line to list all of your open ports you can reduce the number of layers.

By grouping `RUN` commands together intelligently you can share more layers between containers.  Of course if you have a common set of packages across multiple containers then you should look at creating a seperate base image containing these that all of your images are built from.

For each layer that you can share across multiple images you can save a ton of disk space.

## Volume contaimers

If you use Volume containers,  don't bother trying to save space by using a small image,  Use the image of the application you'll be serving data to.  If you do that and `docker commit` the data volume you not only have your data commited to the container, but the actual application as well which is very useful for debugging.

## Cheat

If you've built an image and discover when you run it that there's a package missing add it to the bottom of your `Dockerfile` rather than in the `RUN apt-get` command at the top.  This means you can rebuild the image faster.  Once your image is correct and working you can reorganize your `Dockerfile` to clean such changes up before commiting it to source control.


## Example

A `Dockerfile` for installing graphite would look something like this if it was written like a `bash` script:

```
FROM ubuntu:trusty
MAINTAINER Paul Czarkowski "paul@paulcz.net"

RUN apt-get -yq update

# Apache
RUN \
  apt-get -yqq install \
    apache2 \
    apache2-utils \
    libapache2-mod-python \
    python-dev \
    python-pip \
    python-cairo \
    python-pysqlite2 \
    python-mysqldb \
    python-jinja2
    sqlite3 \
    curl \ 
    wget \
    git \
    software-properties-common

RUN \
  curl -sSL https://bootstrap.pypa.io/get-pip.py | python && \
    pip install whisper \
    carbon \
    graphite-web \
    'Twisted<12.0' \
    'django<1.6' \
    django-tagging

# Add start scripts etc
ADD . /app

RUN mkdir -p /app/wsgi
RUN useradd -d /app -c 'application' -s '/bin/false' graphite
RUN chmod +x /app/bin/*
RUN chown -R graphite:graphite /app
RUN chown -R graphite:graphite /opt/graphite
RUN rm -f /etc/apache2/sites-enabled/*

ADD ./apache-graphite.conf /etc/apache2/sites-enabled/apache-graphite.conf

# Expose ports.
EXPOSE 80 
EXPOSE 2003 
EXPOSE 2004 
EXPOSE 7002

ENV APACHE_CONFDIR /etc/apache2
ENV APACHE_ENVVARS $APACHE_CONFDIR/envvars
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_RUN_DIR /var/run/apache2
ENV APACHE_PID_FILE $APACHE_RUN_DIR/apache2.pid
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_LOG_DIR /var/log/apache2

WORKDIR /app

# Define default command.
CMD ["/app/bin/start_graphite"]

```

However an optmized version of this same Dockerfile based on what was discussed earlier would look like the following:

```
# 1 - Common Header / Packages
FROM ubuntu:trusty
MAINTAINER Paul Czarkowski "paul@paulcz.net"

RUN apt-get -yq update \
  && apt-get -yqq install \
    wget \
    curl \
    git \
    software-properties-common

# 2 - Python
RUN \
  apt-get -yqq install \
    python-dev \
    python-pip \
    python-pysqlite2 \
    python-mysqldb

# 3 - Apache
RUN \
  apt-get -yqq install \
    apache2 \
    apache2-utils

# 4 - Apache ENVs
ENV APACHE_CONFDIR /etc/apache2
ENV APACHE_ENVVARS $APACHE_CONFDIR/envvars
ENV APACHE_RUN_USER www-data
ENV APACHE_RUN_GROUP www-data
ENV APACHE_RUN_DIR /var/run/apache2
ENV APACHE_PID_FILE $APACHE_RUN_DIR/apache2.pid
ENV APACHE_LOCK_DIR /var/lock/apache2
ENV APACHE_LOG_DIR /var/log/apache2

# 5 - Graphite and Deps
RUN \
  apt-get -yqq install \
    libapache2-mod-python \
    python-cairo \
    python-jinja2 \
    sqlite3

RUN \
    pip install whisper \
    carbon \
    graphite-web \
    'Twisted<12.0' \
    'django<1.6' \
    django-tagging

# 6 - Other
EXPOSE 80 2003 2004 7002

WORKDIR /app

VOLUME /opt/graphite/data

# Define default command.
CMD ["/app/bin/start_graphite"]

# 7 - First use of ADD
ADD . /app

# 8 - Final setup
RUN mkdir -p /app/wsgi \
  && useradd -d /app -c 'application' -s '/bin/false' graphite \
  && chmod +x /app/bin/* \
  && chown -R graphite:graphite /app \
  && chown -R graphite:graphite /opt/graphite \
  && rm -f /etc/apache2/sites-enabled/* \
  && mv /app/apache-graphite.conf /etc/apache2/sites-enabled/apache-graphite.conf
```

### 1 - Common Header / Packages

This is our most shareable layer.  All the images running on the same host should start with this.  You can see I've added a few things like `curl` and `git` which while they're not necessarily needed they're useful for debugging and because they're in such a shareable layer,  they don't take up much room.

### 2 - Python, 3 - Apache

Here we get to our language specifications.   I've included the Python and Apache sections here because it's not super clear which should go first.

If we put python first,  then any other image that uses Apache can get a few free python packages,  If we put Apache first then we could have a Ruby app that also includes that layer and get Apache for free ( hell you can just give it python for free anyways ).

### 4 - Apache Envs

I'm calling these out seperately for a few reasons.  

Firstly, they should come either directly directly after the Apache section so that it's easier to make them common ( and cached ) between multiple images.   You might not think it matters since calls like `ENV` are so cheap, but I have seen random `ENV` calls take 10 seconds or so.  If you have a lot, then its good to keep them cached, but you also don't want a changed `ENV` to invalidated the cache of installing Apache.

They're a pretty good example of something you might want to start with at the bottom of your container and move them up higher once you're unlikely to change them again.

Secondly, to mention that I really wish Docker provided a way to specify multiple ENVS on the same line so that I can reduce the number of layers I end up with.

### 5 - Graphite and Deps

This contains some Graphite specific `apt` and `pip` packages.  You could join them into a single command by joining them with `&&` but I kept them seperate so that if `pip` package requirements change it won't need to also reget the `apt` packages.

### 6 - Other

This contains a bunch of cheap commands like `ADD` and `VOLUME` they're probably less likely to change than the previous package installs, but are also cheaper to run, so its less important if their cache is invalidated.

Keep them towards the bottom though as you don't want any changes to them to invalidate the cache for a more costly command.

### 7 - First ADD

You should wait until the last possible moment to use the `ADD` command as any commands after it are never cached.

### 8 - Final setup

I have grouped these final commands into a single layer and they're after the `ADD` commands as they manipulate files that come from the `ADD`

## FIN.

Hopefully this has given you some insight into how to build a better `Dockerfile`.  These are all things I have learned from experience in building my own Docker images and while they may not apply to all situations ( or may be flat out wrong ) they defintely seem to improve my development experience.



