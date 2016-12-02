---
layout: post
title:  "Predections for 2017"
date:   2016-12-2 10:00
categories: java
comments: true
---

The end of the year is coming near, so it's time for some predictions! Note that these are just predictions and a bit of bashing, and very likely to be inaccurate.

Java 9 modules
---

Java 9 will be released, and it will include the Module System (aka Jigsaw). I kind of predicted this last year already, because I started writing a book for O'Reilly together with Sander Mak about Java 9 Modularity in early 2016. Still, last year it was a little bit of a gamble. Jigsaw was still far from done, and looking back at previous Java releases it wouldn't surprise a lot of people if it wouldn't make it in Java 9 either. The current status is that Jigsaw is merged to the main Java 9 code base (until recently it was a separate download) which means it is now extremely likely to make it to the final release. This is great, because the Module System, and the end of the classpath, is a huge step for the platform.

The second part of my prediction is that the second half of the year will be all about the ecosystem around Java 9, with tool integration and support in frameworks. This work is already ongoing, but will unsurprisingly see a big uptake after the Java 9 release.

JavaScript will basically become Java
---

JavaScript is growing up. I'm not so much talking about ES6 language features (although they are great), but more about the ecosystem. Looking at a framework like Angular 2, a code base becomes quite similar to our old-school server side Java code bases. Even more so when using TypeScript with it's annotations. Sure, it's easy to argue this is overkill and unneeded complexity for simple applications. Many modern web applications are not simple however, and need such a framework to keep a large code base maintainable. Looking at the average dependency tree of JavaScript projects it's also a trend that they are size wise comparable to Java projects (this is bad). At the very least, JavaScript is not the less complex alternative for Java anymore.

Container platforms
---

The race between cloud providers is not about virtual machines any more. Strict lock-in PaaS offerings (like AppEngine) will also lose popularity. Instead the race will focus on container platforms, making it as easy as possible to run your containers in the cloud. Kubernetes [1] will play a huge role in this. It is already the basis for Google Container Engine and OpenShift, and Azure has support as well. Cloud providers, including many offerings from smaller companies, will focus on integrating Kubernetes with other parts of the stack; load balancers, storage etc. The great thing for us as developers is that platforms like Kubernetes give a nice abstraction, reducing lock in to a specific cloud provider. 2017 will be the year where hosted container platforms become what IaaS was a few years ago.

Machine learning and big data APIs 
---

The other part of the cloud story will be APIs that make machine learning and big data approachable for small teams. These topics require complex and expensive infrastructure, and very specific knowledge. Higher level APIs offered by cloud providers will make these technologies more accessible. This will be the next way for cloud providers to lock developers into their platform. The availability of these technologies will also be the basis for more interesting applications.


Microservices hype cooling down
---

Ok, maybe just wishful thinking... In 2017 I hope, and to some extend expect, to see more people realise that they don't need a micro service architecture. In 2016 I have seen many using micro services as a synonym for modern light weight frameworks. If you're still stuck using outdated technology and app servers that take forever to startup, of course these light weight alternatives are a great way forward! That doesn't necessarily mean you need distributed architecture as well though. Not just following hypes, there are very valid reasons for distributed/micro services architecture obviously. By joining the company that is probably most famous for this type of architecture (Netflix) I look forward to experiencing this first hand.

For those who do not understand the distinction between architecture and tools, micro services will become the new SOA in 2017 and will likely invest a lot of money in commercial tools they don't actually need.


[1] [One year using Kubernetes in production: Lessons learned](http://techbeacon.com/one-year-using-kubernetes-production-lessons-learned)