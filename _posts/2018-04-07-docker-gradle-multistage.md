---
layout: post
title:  "Building Java Docker images with Gradle and Docker multistage builds"
date:   2018-04-07 12:00
categories: Java
comments: true
---

Building vs running
----
Docker is a great tool for both building and deploying Java applications.
Building a Java application inside a container gives portable builds, you can run the build on any machine that has Docker, and build failures because of environment differences are a thing of the past. The build runs without dependencies of having the correct version of the JDK and Gradle installed.

Running a Java application inside Docker is great as well.
It lets us leverage tools such as Kubernetes, and again, takes away environment dependendencies. 
_Building_ inside a container, and _running_ inside a container have different requirements however. 
To build a Java application, you'll typically need a JDK, and a build tool such as Gradle.
Gradle (and other build tools) provide Docker images that contain an installation of the tool.
When we create an image to run the application, we prefer to strip down the image as much as possible. 
The smaller the image (in terms of megabytes), the better.
This means a lightweight OS, a (potentially stripped down) JRE instead of a full blown JDK, and no extra tools besides the app itself.

Although these requirements seem simple, and apply to almost every programming language, they were not easy to achieve in the near past.
Typically you would need two different Dockerfiles; one to build and one to run, with some volume trickery to get the binary produced by the build into the _run_ image.
Practically, most people wouldn't take the trouble and just end up with a much larger than necessary image.

Docker multistage builds
----

A while ago Docker introduced support for multistage builds.
Basically multiple images defined in a single Dockerfile, and the ability to pass artifacts from one stage to the next.
This is perfect for creating optimal Java images!

The example discussed below is based on the [modular Vert.x application](https://github.com/java9-modularity/java9-vertx) that I used in a [previous blog post](http://paulbakker.io/java/java9-vertx).
It's main feature is the use of the Java module system, and it's now updated to Java 10 (e.g. using the `var` keyword).

The application can be built using Gradle, using it's (still experimental) plugin for the module system.
The application is a multi module Gradle project, where one of the modules (easytext.web) contains the _Application_ plugin, which builds a distribution of the application.
A distribution contains all the JAR files for the application and a shell script to start it.

The `build.gradle` file for the `easytext.web` project is listed below.

{% highlight java %}
plugins {
    id 'java'
    id 'application'
    id 'org.gradle.java.experimental-jigsaw' version '0.1.1'
}

dependencies {
    compile 'io.vertx:vertx-rx-java2:3.5.1'
    compile project(':easytext.pagefetch')
    compile project(':easytext.vertx')
    compile project(':easytext.algorithm.api')

    runtime project(':easytext.algorithm.coleman')
    runtime project(':easytext.algorithm.kincaid')
    runtime project(':easytext.algorithm.naivesyllablecounter')

}

javaModule.name = 'easytext.web'
mainClassName = "javamodularity.easytext.web.Main"
{% endhighlight %}

Creating a multistage Java image
----

Now lets have a look at the Dockerfile that contains the multistage build.

{% highlight java %}
FROM gradle:jdk10 as builder

COPY --chown=gradle:gradle . /home/gradle/src
WORKDIR /home/gradle/src
RUN gradle build

FROM openjdk:10-jre-slim
EXPOSE 8080
COPY --from=builder /home/gradle/src/easytext.web/build/distributions/easytext.web.tar /app/
WORKDIR /app
RUN tar -xvf easytext.web.tar
WORKDIR /app/easytext.web
CMD bin/easytext.web
{% endhighlight %}

First, notice that there are two `FROM` lines in this Dockerfile.
Each `FROM` defines a stage, and a stage can optionally have a name.
The first `FROM` is named `builder`, so that we can reference it in the following stage.
The builder stage is based on the [Gradle Docker image for Java 10](https://hub.docker.com/_/gradle/).
It contains both a Java 10 JDK and a Gradle installation.
All we need to do is add our project files and run the Gradle build.

Because the [Application plugin](https://docs.gradle.org/current/userguide/application_plugin.html), the build produces a tar file containing a distribution of the complete app.

The second stage is based on the [offical OpenJDK 10 image](https://hub.docker.com/_/openjdk/).
We only need a JRE because the application is already built, and we can do without UI frameworks.
This means we can use the recently introduced `jre-slim` Docker image provided by OpenJDK.

The most interesting part about the multistage build is the `COPY --from=builder` line.
This can be used to copy artifacts from one stage to the next.
In this example we take the distribution tar, and copy that to the next stage.
In this stage we untar the distribution and start the application.

Building the image is done using a plain Docker build command:

{% highlight java %}
docker build -t vertx-demo . 
{% endhighlight %}

This produces an image of around 300mb.
Not as good as a Go application, but very useable!

Running the image is a simple Docker run, or more likely, using the container management system of your choice.

{% highlight java %}
docker run -ti --rm -P vertx-demo 
{% endhighlight %}

An even smaller JVM with JLink
----

If you have been reading [our book](https://javamodularity.com/), you might wonder if we couln't further bring the image size down by using a custom JVM created with JLink.
This would definetely be the case, but unfortunately JLink only works when all the application's dependencies are modules.
This is for good reasons, but makes it impractical for an application built on the Vert.x stack right now, since the Vert.x modules don't have `module-info.java` files yet.
However, this will be changing in Vert.x 3.6!