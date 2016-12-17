---
date: "2014-12-22T21:31:03-06:00"
title: "Multi Process Docker Images Done Right"
categories: [ "docker", "devops" ]
---

For some values of 'right'
--------------------------

Almost since [Docker](http://docker.com) was first introduced to the world there has been a fairly strong push to keeping containers to be single process.   This makes a lot of sense and definitely plays into the [12 Factor](http://12factor.net) way of thinking where all application output should be pushed to `stdout` and docker itself with tools like [logspout](https://github.com/progrium/logspout) now has fairly strong tooling to deal with those logs.

Sometimes however it just makes sense to run more than one process in a container,  a perfect example would be running [confd](https://github.com/kelseyhightower/confd) as well as your application in order to modify the application's config file based on changes in service discovery systems like [etcd](https://github.com/coreos/etcd).   The [ambassador](https://docs.docker.com/articles/ambassador_pattern_linking/) container way of working can achieve similar things, but I'm not sure that running two containers with a process each to run your application is any better than running one container with two processes.

<!--more-->

If you're going run multiple processes you have a few options to do it.

1. Start the container with the first process adnd then use the new `docker exec` command to start the second.
2. Start them in sequence in a `bash` script and background all but the last process with a `&` at the end of the line.
3. Use a Process Supervisor such as Supervisord or Runit.


I haven't really messed around with the first option, maybe it could work out, but you'd lose the logs from the second process as it would need to output via the first process' `stdout`.

The Bash Script
---------------

Up until recently the way I have been running multiple processes is via the `bash` script method, but it feels really clumsy and fragile and while it works I've never been particularly fond of it.

Here's an snippet from such a script from my [docker-elk_confd](https://github.com/paulczar/docker-elk_confd) project which builds out the [ELK]() stack using values in `etcd` to orchestrate clustering and configuration via `confd`.

```
echo Starting ${APP_NAME}

confd -node $ETCD -config-file /app/confd.toml -confdir /app &
/opt/elasticsearch/bin/elasticsearch -p /app/elasticsearch.pid &

# while the port is listening, publish to etcd
while [[ ! -z $(netstat -lnt | awk "\$6 == \"LISTEN\" && \$4 ~ \".$PUBLISH\" && \$1 ~ \"$PROTO.?\"") ]] ; do
  publish_to_etcd
  sleep 5 # sleep for half the TTL
done
```

As you can see I've started two processes `elasticsearch` and `confd` both backgrounded and then I finish with a loop which publishes data to etcd every 5 seconds until the `elasticsearch` process quits listening on its published tcp port.  This works, but it leaves me feeling a bit icky.

Process Supervisor
------------------

I have used various supervisors in containers before but never really liked the experience as I could never get all the logs out to `stdout` and using the standard docker logging mechanisms so I've always gone back to the `bash` script method.  Recently while working on the ELK project mentioned above I decided to give using a process supervisor another chance.

My primary measure of success for using a supervisor going forward was to come up with a way to push all output to the supervisor's stdout so that I can use the regular docker logging.

I decided to try with [supervisor](http://supervisord.org) as a starting point because it is a fairly small install and has an easily templatable config.   At about the same time I was looking at this I found a [blog post](http://supervisord.org) ( I believe it was linked in a recent Docker Weekly ) that talked about using `supervisor` in docker containers.  They had even (sortof) solved the logging problem,  however the logging was appended with debug lines and made it messy and difficult to read.  I figured there had to be a cleaner way.

Reading through the documentation I saw that you can specify a file to log each supervised process to.   I just needed a way to hijack that config item to write to supervisor's stdout instead.   Turns out that's quite easy as there's a special device `/dev/stdout` which links to `/dev/self/fd/1` which is the `stdout` for the running application.   I quickly threw together a test and it did indeed pipe the logs from the process through `stdout` of supervisor.

I end up with a `/etc/supervisord.conf` ( which is written out by confd before supervisor is started ) file that looks like this:

```
[supervisord]
logfile=/dev/null
pidfile=/var/run/supervisord.pid
nodaemon=true

[program:publish_etcd]
command=/app/bin/publish_etcd
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
auto_start=true
autorestart=true

[program:confd]
command=confd -node %(ENV_ETCD)s -config-file /app/confd.toml -confdir /app
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
auto_start=true
autorestart=true

[program:elasticsearch]
command=/opt/elasticsearch/bin/elasticsearch
redirect_stderr=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
auto_start=true
autorestart=true
```

and my boot script that docker runs the following to launch my app:

```
echo Starting ${APP_NAME}
/usr/bin/supervisord -c /etc/supervisor/supervisord.conf
```

All output from `Elasticsearch`, `confd`, `supervisord` now output via the docker logging systems so that I can see what is going on by running:

```
$ docker logs elasticsearch
docker logs -f 7270755ce94c03dda930fbdedeee7722dddf6fdbbf8902aaee52c9f94f2147ca
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO /opt/elasticsearch/config/elasticsearch.yml has md5sum 08a09998560b7b786eca1e594b004ddc should be d83b49b485b5acad2666aa03b1ee90a0
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO Target config /opt/elasticsearch/config/elasticsearch.yml out of sync
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO Target config /opt/elasticsearch/config/elasticsearch.yml has been updated
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO /etc/supervisor/supervisord.conf has mode -rw-r--r-- should be -rwxr-xr-x
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO /etc/supervisor/supervisord.conf has md5sum 99dc7e8a1178ede9ae9794aaecbca436 should be ad9bc3735991d133a09f4fc665e2305f
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO Target config /etc/supervisor/supervisord.conf out of sync
2014-12-23T04:46:02Z 7270755ce94c confd[37]: INFO Target config /etc/supervisor/supervisord.conf has been updated
Starting elasticsearch
2014-12-23 04:46:02,245 CRIT Supervisor running as root (no user in config file)
2014-12-23 04:46:02,251 INFO supervisord started with pid 51
2014-12-23 04:46:03,255 INFO spawned: 'publish_etcd' with pid 54
2014-12-23 04:46:03,258 INFO spawned: 'elasticsearch' with pid 55
2014-12-23 04:46:03,260 INFO spawned: 'confd' with pid 56
==> sleeping for 20 seconds, then testing if elasticsearch is up.
[2014-12-23 04:46:04,146][INFO ][node                     ] [Sultan] version[1.4.2], pid[55], build[927caff/2014-12-16T14:11:12Z]
[2014-12-23 04:46:04,149][INFO ][node                     ] [Sultan] initializing ...
[2014-12-23 04:46:04,156][INFO ][plugins                  ] [Sultan] loaded [], sites []
2014-12-23 04:46:05,158 INFO success: publish_etcd entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2014-12-23 04:46:05,159 INFO success: elasticsearch entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2014-12-23 04:46:05,161 INFO success: confd entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

One last thing that I should mention.  the `publish_etcd` talk in the supervisor config is running a script that contains the `while` loop to make sure that `elasticsearch` is listening on the approriate port, If that loop is broken it means that`elasticsearch` is not responding and it sends a kill signal to `supervisor` which then causes the container to shoot itself in the head because the rest of the  processes running are useless without `elasticsearch` running.