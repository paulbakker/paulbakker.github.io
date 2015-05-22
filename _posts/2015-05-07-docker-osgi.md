---
layout: post
title:  "OSGi and Docker, a perfect team"
date:   2015-05-07 22:15:19
categories: Docker
---

Docker makes it easy to run applications or application components on different environments. You don't have to worry anymore about installing environment dependencies such as the correct JVM version are installed on a host system, because they are shipped as part of a container. 

Containers are also very light weight. They are very quick to start and don't add a lot of overhead (in contrast to virtual machines). This opens up a lot of interesting possibilities when looking at scalable production clusters.

Before we can understand how Docker helps in this context, we should first understand how these environments look like in the first place.

Scalable cloud architecture
--

Taking a look at how we host applications, the first important concept is to make cluster nodes disposable. This means that you assume that a node will be replaced by another one at some point, and that you don't care when this happens. The cluster should takes care of replacing unhealthy nodes. This does have an impact on your architecture. First of all, application nodes should be stateless.

Stateless means that the node doesn't keep any state for client sessions or requests it's handling. Of course every application has state, but this should be offloaded to a datastore or distributed cache. If we step into the OSGi world for a moment, you may wonder what this means for the bundle cache. Generally the bundle cache is stored on the file system. This is no problem for a stateless architecture, as long as you don't use the bundle storage for storing application state. In other words; if the bundle cache would be deleted and you restart the app, it shouldn't be a problem.

Having a stateless architecture, we can automate our cluster. We can auto scale, adding or removing nodes from the cluster when load on the application changes. A new node will be registered automatically to the load balancer, and start to take load. When a node is removed or becomes unhealthy, the load balancer will remove the node. Based on this we can add failover as well, nodes can be replaced by new nodes automatically.

If your application is not built for this, you will only get limited value from containerising. 

Docker in cloud infrastructure
--

Containers make achieving infrastructure as described above easier. We can think about containers as processes. Our infrastructure only needs to know how to start containers, it doesn't need to know what runs inside the containers. This also makes it easy to mix and match different technologies, without making the infrastructure more complex.

There are several interesting tools available today to create clusters with automatic scheduling. Scheduling is the process of assigning containers to nodes, start containers on multiple nodes for failover and restarting containers when they crash or when load changes. Kubernetes, Docker Swarm and Mesos are probably the most interesting tools in this area.

Requirements for successful Docker containers
--

When we decide to use Docker and one of the available schedulers as the core of our infrastructure, we still need to package our application components in containers. In our case all application components are developed in OSGi. It turns out to be trivial to run OSGi applications as Docker containers, and the remainder of this post will show how. 

Although not hard requirements, there are a few things that make it easy and convenient to containerise something.

* Self contained application binary
* Accept external configuration (e.g. from environment variables)
* Lighweight, contain nothing more than the absolute minimum to run the app
* Application should be fast to startup
* Run as a foreground process

This can obviously be achieved with a lot of different technologies. OSGi scores really well here though. In the following sections we will see how.

Containerising OSGi apps
--

Docker containers generally run a single foreground process. This means containers are (in general) small and only do a single thing. Bndtools projects come with a Gradle build out of the box. Using this build we can easily generate a self contained, runnable JAR file. A runnable JAR is the perfect candidate to start in a container; it will keep the Docker file simple and easy to understand.

Let's assume we have a Bndtools workspace containing a .bndrun configuration in a project named "run". We can build an executable JAR from Gradle:

{% highlight bash %}
gradle build run:export
{% endhighlight %}

This will generate a runnable JAR in the "run/generated/distributions/executable" folder. In general an OSGi application also needs some configuration, so it's often convenient to add a small Gradle task to copy the runnable JAR and configuration files to a directory.

{% highlight groovy %}
task copyConf(type : Copy) {
 from "run/conf"
 into "release/conf" 
}

task copyRunnables(type : Copy) {
 from "run/generated/distributions/executable"
 into "release"
}

task runnable(dependsOn: ['copyConf', 'copyRunnables']) { 
}
{% endhighlight groovy %}

Now we can build into a "release" folder: 

{% highlight bash %}
gradle build run:export runnable
{% endhighlight bash %}

The next step is to create a Docker container containing our runnable JAR and configuration. This turns out to be very easy, because our runnable JAR can be started with a single command. The only thing we need in the base image is Java, so we use a lightweight Busybox image.

{% highlight bash %}
FROM jeanblanchard/busybox-java:8
#Include the OSGi runnable JAR
COPY release/full.jar /app/full.jar

#Include configuration files
COPY release/conf /app/conf/
WORKDIR /app
EXPOSE 8080
{% endhighlight %}

#Start the runnable JAR
CMD java -jar full.jar

We can now build and run the container:

{% highlight bash %}
docker build --tag amdatu/dockerexample .
docker run -ti --rm amdatu/dockerexample
{% endhighlight %}

If your app includes a shell like the Gogo shell, make sure to pass in the "-ti" arguments, even if you run the container as a daemon. Without a tty the Gogo shell will exit the framework.

Configuration
--

Most applications need configuration such as database connection urls. Configurations are often different for different environments. Our OSGi application of course just uses Configuration Admin, for example using the Amdatu Configurator. Now we only need a way to pass configuration to our container.

We've found three approaches to be convenient, and I will discuss all three.

* Build separate containers for different environments
* Pass configuration as environment variables
* Use a key/value store

Build separate containers for different environments
==

If the number of environments (e.g. dev/test/prod) is limited, it can be an easy approach to build a separate container for each environment that includes the configuration. This way the containers are self contained and don't need any special attention when starting.

Because Docker containers are layered, we can create a layer containing our application binary, and a second layer containing just the configuration files for an environment.

Start by creating a directory structure with the files for each environment:

{% highlight bash %}
- release
- Dockerfile
- prod
--- configfile.cfg
--- otherconfig.cfg
--- Dockerfile
- test 
--- configfile.cfg
--- otherconfig.cfg
--- Dockerfile
{% endhighlight %}

{% highlight bash %}
#Base image containing only the application binary
FROM jeanblanchard/busybox-java:8
MAINTAINER paulbakker
COPY release/full.jar /app/full.jar
{% endhighlight %}

We now use this container as a base image and extend it with configuration.

{% highlight bash %}
#Using our base image above
FROM paulbakker/myapp:base
MAINTAINER paulbakker
COPY *.cfg /app/conf/
WORKDIR /app
EXPOSE 8080
CMD java -jar full.jar
Pass configuration as environment variables
{% endhighlight %}


Using environment variables
==

Statically building images for different environments is of course still not very flexible, it only works if you know all configuration up front. A second approach is to pass in environment variables to your container. Docker makes this easy, you can pass in variables using the "-e" parameter to Docker run.

In OSGi we typically use Configuration Admin to configure components. The Amdatu Configurator is an extension to this, which can read key/value and XML files. These configuration files can contain variables, which can be subsituted by environment variables.

Let's take a property file as an example:

{% highlight bash %}
myproperty=${context.myproperty}
{% endhighlight %}

We can use this property as follows:

{% highlight bash %}
docker run --ti --rm -e "myproperty=myvalue" amdatu/example
{% endhighlight %}

Read more about the Amdatu Configurator [here](http://amdatu.org/components/configurator.html).

Using a key/value store
==

The downside of introducing environment variables is that it can make docker run commands really long. It would be better if the configuration can be stored somewhere externally. In most production Docker environments there is already a key/value store such as etcd or Consul to facility service discovery. Of course we can use this key/value store for our own configuration as well! Both etcd and Consul have HTTP REST apis, so it's trivial to read configuration from a Java application.

For example, for Consul we could create a directory. In this directory we can use the component's PID that we want to configure as keys, and as value a simple key/value file.

{% highlight bash %}
#http://[myconsulserver]:8500/v1/kv/config/org.mycomponent.mypid

myproperty=myvalue
{% endhighlight %}

We can now create a component that reads all keys from the directory over HTTP (http://[myconsulserver]:8500/v1/kv/config/?recurse=true) and pass the values to Configuration Admin.

So are Docker and OSGi really the best companions?
--

Remember that your architecture must be prepared for containerisation first before you can really get the benefits of running in Docker. Before that, there are probably better ways to spend your time. Once you're ready to move to a containerised world however, OSGi makes it very easy to do so.
