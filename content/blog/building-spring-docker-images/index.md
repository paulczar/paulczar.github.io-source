---
date: "2019-01-25"
title: "Building Spring Docker Images"
categories: ["kubernetes","pivotal","spring", "docker"]
---

While investigating running [Spring](https://spring.io) applications on Kubernetes I discovered that a lot of the existing example Spring applications do not have a `Dockerfile` in their git repository. I thought this odd at first (and frankly still do).

What I discovered though, is there's quite a number of ways to build [Spring (and Java in general) container images](https://spring.io/guides/gs/spring-boot-docker/) that don't necessarily rely on writing a Dockerfile.

Full disclosure, I am a firm believe that any opensource project of consequence (where feasible) should ship a Dockerfile in their git repo, and ideally have images up on the Docker hub (or other public container registry) as it allows for newcomers to experience your application or project in just a few seconds with no need to play detective to try and figure out how to get it running.

I will demonstrate building the [Spring Pet Clinic example application](https://github.com/spring-projects/spring-petclinic) into container images.

If you want to follow along at home start by cloning down the repo to your local machine:

```console
git clone https://github.com/spring-projects/spring-petclinic.git

cd spring-petclinic
```

# Option 1 - Dockerfile

The Pet Clinic app uses Maven to build a .jar file, so we have a few options here.

## Build .jar and then copy it into a Java Image

This assumes that you have a suitable version of Java and Maven on your system.

Start by building the project into a .jar file with Maven:

```console
$ mvn install -DskipTests
[INFO] Installing /home/pczarkowski/development/demo/spring-into-kubernetes-1/spring-petclinic/target/spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar to /home/pczarkowski/.m2/repository/org/springframework/samples/spring-petclinic/2.1.0.BUILD-SNAPSHOT/spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar
[INFO] Installing /home/pczarkowski/development/demo/spring-into-kubernetes-1/spring-petclinic/pom.xml to /home/pczarkowski/.m2/repository/org/springframework/samples/spring-petclinic/2.1.0.BUILD-SNAPSHOT/spring-petclinic-2.1.0.BUILD-SNAPSHOT.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 26.984 s
[INFO] Finished at: 2019-01-25T09:23:11-06:00
[INFO] ------------------------------------------------------------------------
```

As you can see this resulted in a Java file `spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar`.  We can create a Dockerfile to ingest this called `Dockerfile.cp`:

```console
FROM openjdk:11.0.1-jre-slim-stretch
EXPOSE 8080
ARG JAR=spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar
COPY target/$JAR /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

> Note: because we already built the Jar we only need a slim JRE image to run it in. We can also use an ARG for the file name in case we need to change it on build with `--build-arg JAR=...`.

A simple `docker build` command should create us an image we can run:

```console
docker build -f ./Dockerfile.cp -t spring/petclinic .
Sending build context to Docker daemon  98.22MB
Step 1/5 : FROM openjdk:11.0.1-jre-slim-stretch
11.0.1-jre-slim-stretch: Pulling from library/openjdk
5e6ec7f28fb7: Pull complete
1cf4e4a3f534: Pull complete
5d9d21aca480: Pull complete
0a126fb8ec28: Pull complete
1904df324545: Pull complete
e6d9d96381c8: Pull complete
Digest: sha256:965a07951bee0c3b1f8aff4818619ace3e675d91cfb746895e8fb84e3e6b13ca
Status: Downloaded newer image for openjdk:11.0.1-jre-slim-stretch
 ---> 49b31a72a85a
Step 2/5 : EXPOSE 8080
 ---> Running in 1aeaae727a80
Removing intermediate container 1aeaae727a80
 ---> a1a1850f8e8f
Step 3/5 : ARG JAR=spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar
 ---> Running in b6faa7c0faa3
Removing intermediate container b6faa7c0faa3
 ---> 2b55681ac9df
Step 4/5 : COPY target/$JAR /app.jar
 ---> dec4f0d56c9d
Step 5/5 : ENTRYPOINT ["java","-jar","/app.jar"]
 ---> Running in f492e1668fff
Removing intermediate container f492e1668fff
 ---> f669afd61b8d
Successfully built f669afd61b8d
Successfully tagged spring/petclinic:latest
```

Start the new container, wait a minute or so (you can watch the logs with `docker logs -f petclinic` if you want), and then test it:

```
$ docker run -d --name petclinic -p 8080:8080 spring/petclinic
a1d51b6f9a47501dfe90f24866e7fb6c82e436323fa4adc09074e8ac7447a1a7

$ curl -s localhost:8080 | head
<!DOCTYPE html>

<html>

  <head>

    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

$ docker rm -f petclinic
petclinic
```

You can look at the resultant image size using `docker images`, if you want to dive deeper you can also use `docker inspect`:

```console
$ docker images spring/petclinic
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
spring/petclinic    latest              f669afd61b8d        41 minutes ago      318MB
```

## Use a multi-stage Dockerfile

If you don't have Java and Maven on your system, or you want to delegate the whole thing to Docker you can utilize a multi-stage Dockerfile to build the .jar file and then copy it into a slim image.

```dockerfile
FROM maven:3.6-jdk-11-slim as BUILD
COPY . /src
WORKDIR /src
RUN mvn install -DskipTests

FROM openjdk:11.0.1-jre-slim-stretch
EXPOSE 8080
WORKDIR /app
ARG JAR=spring-petclinic-2.1.0.BUILD-SNAPSHOT.jar

COPY --from=BUILD /src/target/$JAR /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

Like before we can use `docker build` to build this image, but unlike before we don't need Java or Maven installed locally:

```console
$ docker build -f ./Dockerfile.multi -t spring/petclinic .
...
...
Successfully built ee062471d65c
Successfully tagged spring/petclinic:latest

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
spring/petclinic    latest              ee062471d65c        18 minutes ago      318MB
```

As you'd expect the Docker Image size is the same as the previous build given we effectively did the same thing, build the Jar and then Copy it into a slim image.

# Option 2 - Google JIB

[Jib](https://github.com/GoogleContainerTools/jib) builds optimized Docker and OCI images for your Java applications without a Docker daemon - and without deep mastery of Docker best-practices. It is available as plugins for Maven and Gradle and as a Java library.

Normally you'd add JIB to your maven build via the pom.xml [as shown here].(https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#setup), To kick the tires we can just pass some extra arguments to maven and get the same result.

You can build your image with JIB (you don't even need Docker running!) and ship it straight up to the docker registry by running the following:

> Note: In this example I am using my docker registry username in the image name so that it is uploaded correctly, you'll want to swap out `paulczar` for your own username.

```console
$ mvn compile com.google.cloud.tools:jib-maven-plugin:1.0.0:build -Dimage=paulczar/petclinic:jib -DskipTests
[INFO] Containerizing application to paulczar/petclinic:jib...
[WARNING] Base image 'gcr.io/distroless/java' does not use a specific image digest - build may not be reproducible
[INFO]
[INFO] Container entrypoint set to [java, -cp, /app/resources:/app/classes:/app/libs/*, org.springframework.samples.petclinic.PetClinicApplication]
[INFO]
[INFO] Built and pushed image as paulczar/petclinic:jib
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 57.478 s
[INFO] Finished at: 2019-01-25T11:03:25-06:00
[INFO] ------------------------------------------------------------------------
```

> Note: This provides a warning `build may not be reproducible`. You can pass an argument to use your own base Java image to make it more deterministic by adding `-Djib.from.image=openjdk:11.0.1-jre-slim-stretch`.

If you want JIB to build against your local docker install and not push the image to the registry you can run the following:

```console
$ mvn compile com.google.cloud.tools:jib-maven-plugin:1.0.0:dockerBuild
...
...
[INFO] Built image to Docker daemon as spring-petclinic:2.1.0.BUILD-SNAPSHOT
[INFO] Executing tasks:
[INFO] [==============================] 100.0% complete
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 10.485 s
[INFO] Finished at: 2019-01-25T11:14:21-06:00
[INFO] ------------------------------------------------------------------------

docker images spring-petclinic:2.1.0.BUILD-SNAPSHOT
REPOSITORY          TAG                    IMAGE ID            CREATED             SIZE
spring-petclinic    2.1.0.BUILD-SNAPSHOT   79d677deeedb        49 years ago        164MB
```

> Note: This image is much smaller than the rest, this is because by default JIB creates a distroless Java image. This might seem like a good idea for the size, but will like the warning from the previous build give you an image that may not be reproducable. I recommend always using the `-Djib.from.image=openjdk:11.0.1-jre-slim-stretch` argument to choose your upstream Java image which will again give you a 318Mb image like the previous builds.

# Others

I've shown you what I believe are the best methods for building a Docker image for your Spring application. There are some other maven plugins that do the same thing:

* [Spotify/dockerfile-maven](https://github.com/spotify/dockerfile-maven) builds a Jar and then uses a user provided Dockerfile to copy it in.
* [spotify/docker-maven](https://github.com/spotify/docker-maven-plugin) builds the whole image for you much like JIB.
* [fabricate/docker-maven](https://github.com/fabric8io/docker-maven-plugin) also builds the whole image like JIB.

# Conclusion

Hopefully after reading this you have a better idea how to build Docker images for your Spring (or general Java) Application. Personally I prefer the multi-stage Dockerfile as your Dockerfile becomes the contract on how your image is built, however I do really like the way i can use JIB to build an image without needing Docker as this simplifies my build environment and means I can very easily use tools like [Travis CI](https://travis-ci.org) or [Drone](https://drone.io) to build my images for me.
