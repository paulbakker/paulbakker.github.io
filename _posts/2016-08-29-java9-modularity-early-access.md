---
layout: post
title: "Java 9 Modularity: O'Reilly Early Access Release" 
category : java 
tags : [java, book, modularity]
---

Early this year my colleague [Sander Mak](http://branchandbound.net/) and I started working on the first drafts of Java 9 Modularity.
We're proud to announce that the firsts bits of what will become the final book are now publicly available!

You can now order the Early Access release of our upcoming book through O'Reilly's webshop: [http://shop.oreilly.com/product/0636920049494.do](http://shop.oreilly.com/product/0636920049494.do).
It is also available on [Safari Books](http://my.safaribooksonline.com/book/programming/java/9781491954157) online, if you have a subscription there.
If prefer reading the final edition on real paper, you can even pre-order [at Amazon](http://amzn.to/2buO9bZ) already.

![Java 9 Modularity Early Access Release](/pics/java9mod_earlyaccess.jpg)

When you order the Early Access release, obviously you will get updates on the content as they come.
No worries, there's more in the pipeline already.
Be sure to follow [@javamodularity](https://twitter.com/javamodularity) if you're interested in updates, or to give us - much appreciated! - feedback.

As of now, the first three chapters are available.
If you happen to be at JavaOne this year, mark your calendar for Tuesday September 22nd, 1:30-2PM (PDT).
We will be at the O'Reilly booth to sign printed copies of the early access release.
Come by and say hi!

### What's in there?
So, what's all the fuss about?
Java's upcoming module system has been many years in the making.
With Java 9 it's finally coming to fruition, and that's a big deal.

On the one hand, it allows new ways of creating strictly more modular applications.
The module system introduces a stronger notion encapsulation into the platform and allows for explicit dependencies between modules.
On the other hand, it's a logical continuation of Java's existing features and philosophy on large-scale software development.
Access control (private/protected/public), programming to interfaces and so on were, and still are, important enablers of modular application development.
The module system takes these features to next level.

The first chapter will take you from how Java works today, to how modules address pain points in the current situation (classpath hell, anyone?!).
Then, the second chapter continues with an introduction of how the JDK itself was modularized using the very same module system application developers can use.
Meanwhile, you'll gain an understanding of the most important concepts of the Java module system.
After reading those two chapters you know why and what, so in the third chapters it's time for the how.
Here, you'll learn how to create your own modules and work with them in practice.

### What can you expect?
Of course, it doesn't end there (although it does end there, for now, in the early access release).
The outline shown at the [O'Reilly site](http://shop.oreilly.com/product/0636920049494.do) is  somewhat outdated already, but gives a good indication nevertheless.

In subsequent chapters we discuss how modules and interfaces together do not fully solve the problem of decoupling consumers and providers of services.
One of the solutions lies in the [Services and ServiceLoader](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html) model that has been updated in Java 9 to work with modules.

The next chapters look at so-called _modularity patterns_.
Existing wisdom, viewed in the light of the new Java module system.
In these chapters, some of the more advanced APIs of the module system will be discussed as well.

Currently, a big focus area for us is migration of existing codebases to Java 9.
Fortunately, the Java module system is designed with migration in mind.
That doesn't mean it will be a walk in the park in all situations.
The book gives practical advice on how to migrate in several steps, both for application developers and library authors.

Last, you can expect an overview of how the Java tooling eco-system interacts with Java 9 modules.
Since this is very much in flux at the moment, those chapters will probably be among the last to be pushed out in the early access release process.

### Moving on
The process of writing Java 9 Modularity is quite challenging, since the final specifications of the module system have not been nailed down yet.
Even before the Early Access release we already had some re-writing to do based on developments in the [JDK 9 Jigsaw](https://jdk9.java.net/jigsaw/) prototype.
We are very much indebted to Alan Bateman and Alex Buckley from the Oracle JDK team, who have been graciously reviewing our work and provided valuable feedback.
Obviously Paul and I will stay on top of any developments in the period until the specification is stable, which is hopefully not too far off.

We are writing this book because we truly believe modular development is an enabler of both agile and reliable application development.
Through this book, we want to show how you can take advantage of age-old principles of modularity in the shiny new version of Java.
Please let us know what you think of the [Java 9 Modularity Early Access](http://shop.oreilly.com/product/0636920049494.do) release.
It's through feedback of early adopters like you we can shape the book into a useful resource for the whole Java community.

The final release of the book is still set for February 2017, right ahead of the Java 9 GA release.
Meanwhile, follow [@javamodularity](https://twitter.com/javamodularity) for interesting news about the Java 9 module system and updates on the book!