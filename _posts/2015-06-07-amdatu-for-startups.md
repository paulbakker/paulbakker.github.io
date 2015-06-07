---
layout: post
title:  "Why you should use Amdatu for you next startup"
date:   2015-06-07 22:31
categories: Amdatu
---

In the past few months I have been working on a side project named [Workouttraxx](https://workouttraxx.com). Workouttraxx is an online signin system for CrossFit boxes and is being used by a few boxes for several months already.

This week we officially launched the product and are now open for CrossFit boxes all around the world to use it, which makes it a perfect time to reflect on the technology we used to build all of this.

Requirements for a productive stack
==

Workouttraxx is a relatively small project. So far most of the development was done by myself and I worked on it in my free time. Compared to the projects I'm normally involved in this is very small. A development stack for such kind of projects must be, first and foremost, highly productive. Typically these kind of projects seem to fit well with frameworks such as Grails and NodeJS, but we'll see that Java/OSGi/Amdatu is an even better choice.

For Workouttraxx we needed the following:

* A modern web UI
* RESTful web services
* Mongo integration
* Scheduling
* Email
* An easy way to deploy

As we will see in more detail in this post, all of this is covered by [Amdatu](http://amdatu.org). Working with OSGi and Amdatu has some other great benefits besides that:

* Extremely fast coding experience
* Modularity with very limited development cost

The fast coding experience we get with Bndtools is really key for why this is such a productive stack. This is exactly the reason a lot of developers prefer working with dynamic languages; it sucks to wait for a build or deployment every time you change some code. With Bndtools we have our code changes available instantly like when working in a dynamic language environment, combined with the benefits of working with a statically compiled language.

<iframe src="https://player.vimeo.com/video/83374139" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

Be warned though, once you get used to this workflow, you're not going to have a lot of fun going back to Maven/Gradle projects... ;-)

Modern web apps
==

To build a modern web application we don't really have to look at traditional Java web frameworks. A JavaScript framework such as AngularJS combined with RESTful web services is all we need. Angular is a great choice, and even more so when combined with TypeScript. All the web development is done completely outside of our Java development environment. Eclipse is simply not very good at working with JavaScript etc., so we primarily use WebStorm. For TypeScript compilation, minification etc. we use Grunt as a build tool when creating production builds. 

Only when we build the final deployable to put on a server we run a Gradle build that includes the minified web resources in an OSGi bundle. This way we only care about a single deployable, but you could separate them completely and host the web resources on a CDN.

Building a backend
== 

The backend is completely built out of OSGi services. This is as easy as putting some [Dependency Manager](http://felix.apache.org/documentation/subprojects/apache-felix-dependency-manager/tutorials/working-with-annotations.html) annotations on your classes.

Even for a relatively small application we want to keep an eye on modularity. Related functionallity should be provided by a single service, and that service should strictly only be responsible for a single thing. For larger code bases this becomes even more important, but even for something reasonably small maintainability can quickly become a nightmare when there are no clear module boundaries. Modularity is of course a design concept, but OSGi and Bndtools make it extremely cheap to do the right thing. Cheap, in this case, means that it doesn't really cost any extra development time to introduce separate modules, so there is really no reason _not_ to do the right thing.

Besides building services and modules we of course need libraries. OSGi is just our core modularity framework, but on top of that we need libraries to create RESTful web services, work with datastores etc. Amdatu provides these kind of components for OSGi, and contains everything you need for building a modern web application.
We can use these components by simply adding the components we would like to use to our runtime. There is no core framework, or a dozen dependencies to add like you often find for other Java frameworks. The reason that this is possible is that OSGi makes the creation of self contained, re-usable libraries possible. 

![Components](../../images/posts/components.jpg)


Integration testing is supported out of the box with Bndtools, which makes it very easy to test our own components in a real OSGi runtime. 

Deployment
==

When launching a new product there are always bugs to fix and improvements to be made. This means deploying new versions very often, and this should be without a lot of effort.

We're running our servers on [Digital Ocean](http://digitalocean.com). A cloud provider that I really started to like. Their services are much simpler than Amazon AWS or Azure, basically the only thing they offer are virtual servers, but you get a lot of performance for very little money. If you don't need the additional services provided by other cloud providers, this is a great place to look.

For our Mongo cluster we decided to run on [Compose.io](http://compose.io) which turns out to be a great service. Our Mongo deployment is relatively small, but we still need a replica set to get failover. Setting this up yourself isn't rocket science, but you would have to take care of running these servers, backups and updates. 

To deploy the application itself we package it in a Docker container. As I wrote in a [previous post](http://paulbakker.io/docker/docker-osgi/), this is very easy when working with a Bndtools project, because we can generate a runnable JAR from our build. The server runs the Docker container behind an Nginx proxy that takes care of SSL offloading and hosting the product website. Updating the application is now as simple as pushing a new Docker container and restart the container at the server. 

This deployment approach works very well for small deployments of a few servers. When looking at much larger workloads, and more demands on monitoring and logging, you would probably want to use something like our commercial Amdatu RTI stack.

