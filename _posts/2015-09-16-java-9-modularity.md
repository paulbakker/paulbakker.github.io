---
layout: post
title:  "Java 9 modularity"
date:   2015-09-16 22:15:19
categories: Java
comments: true
---

At JavaZone, Sander Mak and me gave a talk about modularity in Java 9. The recording of this talk can be found online and is a great start to learn about things to come. Shortly after the presentation some big news was posted on the mailing list: A working prototype of the module system. This blog will go into the prototype and discuss choices made in the design of the module system.

<iframe src="https://player.vimeo.com/video/138736736" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

Modularity in Java has a long history of not making it into releases. With a working prototype available in the recent JDK 9 build, it really looks like it's finally getting close now. Realistically, time is still running very short. At the time of writing this article there are only about three months left until the planned code freeze for Java 9, which is very short to evaluate something as important as the module sytem. Let's ignore that for now, and focus on the prototype.

Goals of Java 9 modularity
--

There are three topics in the modularity story for Java 9:

* Modularize the JDK and JRE
* Hide platform internals (e.g. sun.misc)
* A module system for application developers

While the modularisation of the JDK and hiding platform internals offer a lot of benefits, this blog only goes into the module system that we will use as application developers. A module system is great news; modularity as an architectural principle helps creating maintainable code. In a modular code base changes to implementation code are much better isolated, making it easier to reason about changes. Modularity prevents spaghetti code. To accomplish writing a modular code base we need proper tools however. Of course there's already OSGi, but making modularity part of the language itself will at the very least increase awereness about this topic to more developers.

Diving into the module system prototype
--

Modularity starts with defining module boundaries. A module can contain APIs that should be shared with other modules, and implementation code that should be available only to the module itself. Hiding implementation code to other modules is step one towards decoupling. Practically this makes it possible to change the internals of a module, or even replace it's entire implementation, without even touching other modules. Of course, as long as you keep the module's interface, it's contract, intact. In OSGi we have been doing this for a long time, and it proves to be an extremely important tool to be explicit about which code is, and which isn't public.

Looking at Java 9, a module is defined by a module descriptor "module-info.java". The descriptor contains the name of the module, and defines which packages should be _exported_. An exported package is available to other modules, that _require_ the module that exports the package. Packages that are not explicitly exported, are never visible to other modules, not even when using reflection. 

{% highlight java %}
module com.foo.bar {
    exports com.foo.bar.alpha;
    exports com.foo.bar.beta;
}
{% endhighlight java %}

To use another module's exported package the module must explicitly _require_ that module. Being explicit about a module's dependencies is another piece of the modularity puzzle. This is useful information to a developer, but is also used by the runtime to check if any required modules are missing. This is infinetly better than working with the flat classpath that we have today which only throws ClassNotFoundExceptions at runtime; when some unlucky user triggers a code path that loads a missing class.

{% highlight java %}
module com.foo.baz {
    requires com.foo.bar;
}
{% endhighlight java %}

There is a significant difference with OSGi. In OSGi a module typically _imports_ (which is the same as _requires_) packages, not modules. We have seen that Java 9 modules _export_ packages, but _require_ modules. Depending on modules instead of packages is possible in OSGi as well, but is considered a bad practice. When we depend on a package, we don't care where it comes from, which gives flexibility when it comes to packaging. When depending on a module, there's coupling to the packaging of another module. We can't replace that module by another module that exports the same package, without changing all the module descriptors that depend on it. In OSGi we have learned that this can lead to very inflexible systems. Although requiring modules stays a little closer to the dependency management we know today in most build tools, I think it's a bad choice to do this, but one that we can live with.

Another interesting aspect is that the module-info.java doesn't provide versioning information. This is left to build tools and containers to take care of. I would have strongly preferred to make this part of the modules itself. The "same" module can now potentially mean different things when using different build tools. Also, I'm afraid it will limit compatibility between build tools. This would have been an excellent time to introduce semantic versioning to Java modules, which makes it much easier to reason about versions and the module graph that it produces.

Services
--

When implementation code is hidden to other modules, which probably includes your Dependency Injection framework of choice, a new question emerges. How do you instantiate implementation code which is not visible? How to use implementation code of a module when we can only see it's exported interface?

In OSGi we have the concept of _services_ for this. A module can register an implementation of an inteface as a service to the Service Registry. Other modules can use this implementation by asking the Service Registry for an implementation of the exported interface. The Service Registry returns the service instance, and the module can use this instance without even knowing what module provided this implementation. Exactly the decoupling we are looking for, while very easy to use based on a dependency injection framework that works with the Service Registry.

Java 9 introduces this services concept to Java modules as well. A module that provides a service declares this in module-info.java.

{% highlight java %}
module com.foo.bar {
    requires com.foo.api;
    provides com.foo.api.MyInterface with com.foo.impl.MyImpl;
}
{% endhighlight java %}

A module that want to use a service must add a _uses_ declaration in it's module-info.java.

{% highlight java %}
module com.foo.baz {
    requires com.foo.api;
    uses com.foo.MyInterface;
}
{% endhighlight java %}

The only thing shared by both modules is the API provided by "com.foo.api", in this case a separate module. There is no coupling at all from the "com.foo.baz" module to "com.foo.bar". Using this module definition we can now use the service loader API to get access to the service(s) provided by modules. 

{% highlight java %}
ServiceLoader<com.foo.api.MyInterface> services = ServiceLoader.load(com.foo.api.MyInterface.class);
for(MyInterface i : services) {
	//use service
}
{% endhighlight java %}

That doesn't exactly look like a user friendly way to work with services! This is true, but it is to be expected that dependency injection frameworks will provide this in more friendly ways. In OSGi there is also a, very hard to use, low level service API, but there are several dependency injection frameworks built on top of this that offer service injection by annotations or easy to use Java API.

There is one very big difference with OSGi however. Services in OSGi are inherently dynamic; they can come and go at any time during runtime. This makes it possible to add new services at runtime. From one hand this complicates the programming model significantly (which is bad), on the other hand these dynamics bring us powerful features such as [hot code reloading](https://vimeo.com/user17212182/review/83374139/1cea4e1d61) during development and plugin systems. Services in Java 9 are not dynamic. This keeps the model simple, but requires additional layers of tools and frameworks to provide features still often required by Java developers. This is a tradeoff, there is no right or wrong, but it's an important choice.

The road to adoption
--

Although we now have a prototype, I'm afraid it's going to take some time before we see libraries and frameworks support being used in a modular context. Modular design isn't easy, and migrating something that wasn't designed to work in a modular runtime to become modular can be very challanging. Many developers have complained about how difficult it is to OSGi. Looking at common problems with OSGi it's easy to realize that the problems are caused by design that isn't modular, not so much by the framework. Many of those problems will also become visible when migrating to Java 9 modules. Although adoption might be slow, the fact that modules will be part of the Java runtime will in the end push library and tool authors to be serious about modularity. 

Java 9 vs OSGi?
--

Another obvious question is what place OSGi will still have if modularity becomes part of the Java platform. As much as I would like to see things move faster for Java 9, I think OSGi is still the only viable choice in at least the next two years, probably even longer. First it will still take quite some time before Java 9 is released, but that's only step one. For most non-trivial applications we need frameworks, libraries and tools and as discussed above it will take some time to move this eco system to a more modular state. Until that happens, I strongly recommend using OSGi and [Amdatu](http://amdatu.org) to build applications. That being said, Java 9 is great news in the long term. Modularity is an architecture principle. The closer the tools are to the language, the easier it gets. Although there are some disputable (not to be confused with "wrong") choices in the Java 9 module system, it looks very useable. 

Modularity in a micro services world
--

Is modularity still relevant in a micro services world? Isn't Java modularity delivered at a point where we don't need it anymore anyway? These are questions that I heard several times in the past few days. I think the answer is that modularity is definetly still relevant! The question is valid, when we have many relatively small code bases instead of one gigantic monolithic code base, the need for modularity is smaller. Most systems today are not based on a micro services architecture however, and probably shouldn't be either. A micro services architecture comes at a huge cost, which is justifyable in some cases, but shouldn't just be used as a way to achieve modularity. A module system, like OSGi or Java 9 modules offers the tools to keep large code bases clean. 

Getting started
--

If you want to try the module system prototype for yourself, my collegue [Sander Mak](https://twitter.com/sander_mak) wrote an excellent [getting started guide](http://branchandbound.net/blog/java/2015/09/java-module-system-first-look/) to help you set things up. 

Also make sure to read the official [spec](http://openjdk.java.net/projects/jigsaw/spec/sotms/) of the module system for more details.