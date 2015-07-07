---
layout: post
title:  "Amdatu Kubernets API for OSGi"
date:   2015-07-06 22:15:19
categories: Kubernetes, Amdatu
comments: true
---

We are working a lot with Kubernetes lately and were looking for a Java API to integrate our dashboards with Kubernetes. There used to be a few libraries to choose from, but most are not actively maintained and are still using the depricated beta APIs.

The library which is part of [Fabric8](http://fabric8.io/guide/javaLibraries.html) seems to be the best/only option, but unfortunatly the library's dependency graph is huge mess. This is always bad, you should be very weary if Maven/Gradle tries to pull in half of Maven central, but it's even worse when working in a modular environemnt like OSGi. We started out as we always do with these kind of libraries: wrapping them in a self contained bundle to at least hide the mess. This didn't work out very well, there is just no end to the list of (transitive) dependencies.

So we decided to build something ourselves, and decided to open source it as part of Amdatu. As with every new Amdatu component it will first live in the "labs" incubator until we decide it's stable and useful enough to make it to [amdatu.org](http://amdatu.org). 

Introducing Amdatu Kubernetes
--

The Amdatu Kubernetes API makes it easy to use Kubernetes from OSGi or Java in general. 

{% highlight java %}
@Component
public class Example {
	@ServiceDependency	
	private volatile Kubernetes m_kubernetes;

	public void test() {
		kubernetes.listNodes().subscribe(nodes -> {
			nodes.getItems().forEach(System.out::println);
		});
	}
}
{% endhighlight %}


In a non-OSGi Java application you pass the Kubernetes API url to the constructor of the KubernetesRestClient.

{% highlight java %}
public class PlainJavaExample {
	private Kubernetes m_kubernetes;
	
	@Before
	public void setup() {
		m_kubernetes = new KubernetesRestClient("http://[kubernetes-api]:8080");
	}
	
	@Test
	public void listNodes() {
		TestSubscriber<NodeList> testSubscriber = new TestSubscriber<>();
		m_kubernetes.listNodes().subscribe(testSubscriber);

		testSubscriber.assertCompleted();
		testSubscriber.assertNoErrors();
	}
}
{% endhighlight %}

The API is based on [RX Observables](https://github.com/ReactiveX/RxJava). This makes it easier to deal with the asynchronous nature of the API. 

The model classes used by Amdatu Kubernetes are from the Fabric8 project. These are generated from the JSON schema provided by Kubernetes, so there was no point in re-doing this.

Learn more
--

The [integration tests](https://bitbucket.org/amdatulabs/amdatu-kubernetes/src/6850e7850f171ded7a814473e21128078791d929/org.amdatu.kubernetes.test/src/org/amdatu/kubernetes/test/KubernetesTest.java?at=master) in the project are the best place to learn how to use the API. Configure the test suite with a Kubernets API url and it will create a test namespace with a ReplicationController and Pods and executes several operations on them.

Of course also take a look at the [JavaDocs](https://amdatu.atlassian.net/builds/browse/KUBER-MAIN/latest/artifact/JOB1/javadoc/index.html).

The Kubernetes API is not (yet) fully covered by Amdatu Kubernetes, but the most important methods are, including watches. Of course pull requests are welcome!