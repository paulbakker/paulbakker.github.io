---
layout: post
title:  "Vertx on OSGi, an even better Vertx"
date:   2016-01-10 22:15:19
categories: OSGi
comments: true
---

Always eager to play with other/new/cool technology I spent some time learning about Vertx. Because Vertx is designed to be a general purpose toolbox, I wanted to run it as part of an OSGi application. This turns out to work very well, and this post gets into all the details. The complete example project can be found on [GitHub](https://github.com/paulbakker/osgi-vertx-demo).

Why Vertx, and why Vertx on OSGi?
-----

Vertx is a toolbox for non-blocking, asynchronous modern (web) applications. These characteristics are becoming more important for today's applications so it makes a lot of sense looking at a framework/toolbox like this. I won't go into the discussion if Vertx is the _best_ toolbox for this, but at least it's one of the popular choices.

So why OSGi? OSGi makes creating modular code bases a lot easier, which is the most important reason to use OSGi. Besides that, the dynamic nature of OSGi allows for a development workflow without waiting for builds or restarting the application, which makes coding just so much more fun. Having OSGi for these core concepts combined with Vertx as a non-blocking toolkit should be a great combination! 

The application
-----

The example app that I worked on is a chat application, where the Vertx event bus is used to communicate with browsers, and chat messages are persisted in a Mongo database. Also, a RESTful web service is available to list the chat history. A simple, but not trivial application that requires several modules of Vertx, and has enough separate parts to demonstrate modularity. Besides just getting the application to work, the main goal of the example is to demonstrate how Vertx can fit in very nicely with the dynamic service programming model that we're used to in OSGi. Because the codebase is small you might argue that modularity at this level is really not necessary, and of course that's true. The setup of the code is prepared for being used in much larger code bases, where this _does_ matter a lot.

Just to make the experiment even more interesting, I used the Rx-ified APIs of Vertx.

The front end is built using Angular 2 and using the Vertx SockJS bridge to directly connect to the event bus.

Application Structure
------

The following is a list of bundles (modules) which make the application. Looking at the [GitHub repo](https://github.com/paulbakker/osgi-vertx-demo) you see that each bundle comes from it's own project. Our project structure maps directly to the runtime structure, which makes it much easier to understand. The workspace is created in Bndtools, a plugin for Eclipse to make OSGi development easy. Out-of-the-box we also get a Gradle build for each Bndtools workspace, which we use on build servers. During development, we don't need to run Gradle however.

- chat.api : API only project, containing the ChatMessage class.
- chat.vertx.bootstrap : Wraps Vertx Web and vertx RX and makes the Vertx and Router instance available as OSGi services. 
- chat.vertx.http : Setup static file routing and starts HTTP server.
- chat.vertx.messaging.storage.mongo : Stores chat messages in Mongo
- chat.vertx.messaging.storage.rest : Makes the stored chat messages available in a RESTful web service
- chat.vertx.messaging.storage.itest : OSGi integration test for the Mongo service.
- chat.vertx.mongo : Sets up a MongoClient based on Configuration Admin configuration and publishes it as an OSGi service.
- chat.vertx.sockjs : Sets up the SockJS event bus bridge.
- run : Contains the configuration files, static web resources and bndrun configuration to start the application. Check the chat.bndrun file for the full list of bundles to run the application. The application can also be started from this file directly.

Making Vertx work with OSGi
-----

Vertx itself is very modular. The core, which contains things like the event bus, and low level networking is already a fully functional OSGi bundle. This is great! The basic functinallity works out of the box in OSGi.

Vertx Web, and Vertx RX are not OSGi bundles however. This doesn't have to be a problem. A common way to work around this problem is wrapping the library and it's dependencies inside the OSGi bundle that uses it. That's exactly what the chat.vertx.bootstrap bundle is doing. The project contains a "lib" folder with the JAR files required for Vertx web and Vertx RX. This includes required dependencies. Dependencies that are available as OSGi bundles, like Netty, are not wrapped. This is a choice, and it has pros and cons. 

Vertx Web has lots of optional dependencies. Vertx uses dynamic class loading to test if certain optional jar files are available or not. This way Vertx itself becomes more pluggable. If we would just wrap Vertx Web, these dependencies would become required. When classes are used in an OSGi bundle, these classes need to be imported in the Import-Package header. This guarantees that all used classes are available at runtime, but doesn't match very well with the optional class loading model in Vertx. In a pure OSGi world we would have better ways to do this kind of modular loading, but from a Vertx point of view this model is very reasonable.

Luckily we can easily work around this issue in OSGi. We can make an Import-Package optional as well. If, at runtime, the imported class is exported by another bundle it's imported, otherwise it's ignored. Exactly what we need! The following is a snippet from the [bnd.bnd](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.bootstrap/bnd.bnd) file that makes Groovy an optional dependency.

```
Import-Package: \
   groovy.lang.*;resolution:=optional
```

The chat.vertx.mongo project does the wrapping for Vertx Mongo, and adds some extra functionallity as well as we'll see later.

Vertx as OSGi services
------

With Vertx web nicely wrapped we can just use the Vertx and Vertx Web APIs as usual. In OSGi we have a dynamic service model, including dependency injection, that could make it even better, so that's the next goal.

Usually a Vertx application uses a single _Vertx_ instance. For Vertx Web we also need an instance of _Router_. By making both available as OSGi services, we can use OSGi dependency injection (using Apache Felix Dependency Manager) to easily work with these APIs in OSGi components. This is done in the [Activator](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.bootstrap/src/chat/vertx/bootstrap/VertxActivator.java) class in chat.vertx.bootstrap. 

{% highlight java %}
package chat.vertx.bootstrap;

import org.apache.felix.dm.DependencyActivatorBase;
import org.apache.felix.dm.DependencyManager;
import org.osgi.framework.BundleContext;

import io.vertx.rxjava.core.Vertx;
import io.vertx.rxjava.core.eventbus.EventBus;
import io.vertx.rxjava.ext.web.Router;

public class VertxActivator extends DependencyActivatorBase{

	private volatile Vertx vertx;
	
	@Override
	public void init(BundleContext arg0, DependencyManager dm) throws Exception {
		vertx = Vertx.vertx();
		
		dm.add(createComponent().setInterface(Vertx.class.getName(), null).setImplementation(vertx));
		dm.add(createComponent().setInterface(EventBus.class.getName(), null).setImplementation(vertx.eventBus()));
		
		Router router = Router.router(vertx);
		dm.add(createComponent().setInterface(Router.class.getName(), null).setImplementation(router));
	}

	@Override
	public void destroy(BundleContext context, DependencyManager manager) throws Exception {
		vertx.close();
		vertx = null; 
	}
}
{% endhighlight %}

In a component in another bundle, such as the [chat.vertx.http.HttpServer](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.http/src/chat/vertx/http/HttpServer.java) which bootstraps the static file serving or the [chat.vertx.messaging.rest.MessageHistoryResource](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.messaging.storage/src/chat/vertx/messaging/rest/RoomsResource.java) we can inject those services to work directly with the Vertx (Web) APIs.

{% highlight java %}
package chat.vertx.messaging.rest;

import java.util.HashSet;
import java.util.Set;

import org.apache.felix.dm.annotation.api.Component;
import org.apache.felix.dm.annotation.api.ServiceDependency;
import org.apache.felix.dm.annotation.api.Start;
import org.apache.felix.dm.annotation.api.Stop;

import chat.vertx.messaging.storage.MessageStore;
import io.vertx.core.json.Json;
import io.vertx.rxjava.ext.web.Route;
import io.vertx.rxjava.ext.web.Router;

@Component
public class MessageHistoryResource {
	
	@ServiceDependency
	private volatile Router router;

	private final Set<Route> routes = new HashSet<>();
	
	@ServiceDependency
	private volatile MessageStore messageStore;
	
	@Start
	public void start() {
		Route route = router.get("/messages");
	    routes.add(route);
	
		route.handler(routingContext -> {
			messageStore.listMessagesAsJson().subscribe(
					messages -> routingContext.response().putHeader("content-type", "application/json; charset=utf-8").setChunked(true).write(Json.encodePrettily(messages)), 
					e -> { 
						routingContext.fail(500);
						e.printStackTrace();
					},
					() -> routingContext.response().end());
		});
		
	}
	
	@Stop
	public void stop() {
		routes.forEach(r -> r.remove());
	}
	
}
{% endhighlight %}

Notice that the code above contains a _stop_ method. This is a lifecycle method which is invoked when the component gets de-activated, for example when the bundle is stopped or updated. It's important to clean up correctly in these lifecycle methods. The Vertx API is fully prepared for this; everything that is created can also be disposed.

Modularizing the code
-----

Looking at the example code, there are two levels op modularization going on (which is usual). First, different technical responsibilities are isolated in separate bundles. The chat.vertx.bootstrap bundle _just_ takes care of exporting the required APIs, and making them available as OSGi services. The chat.vertx.http bundle _just_ takes care of setting up static resource serving and starting the http server. Chat.vertx.sockjs only sets up the SockJS bridge and chat.vertx.mongo only creates the MongoClient as a service. This makes it very easy to make technical changes to any of these features, replace them entirely or just stop using them without needing to delve into a large piece of code. The second level of modularity is the separation of funtional areas. The chat.vertx.messaging.storage.mongo and chat.vertx.messaging.storage.rest bundles both just do a single thing, and are fully de-coupled from other functionallity doing different things. They are also nicely abstracted from the exact technical setup of the Router, Eventbus and MongoClient. I did not try to abstract away the Vertx APIs themselves (our functional bundles still use Router etc.). Although technically possible, this is an abstraction too much in my opinion.

Having such a modular setup makes it much easier to grow the code base without ending up with a big ball of mud. The structure also doesn't change much when we add more funtionallity. The technical bundles probably don't even change (we might add a few more), and for new features we just create more bundles. Basically, every funtional domain, or every feature get's it's own OSGi service contained by it's own OSGi bundle. 

Configuration Admin
-----

The MongoClient needs to be configured with an url and database name. Although Vertx does have it's own configuration mechanism, I used the standard OSGi Configuration Admin service to provide this configuration. There are many adapters to work with configuration coming from different sources, which makes it very widely usable. 

The [MongoService](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.mongo/src/chat/vertx/mongo/connection/VertxMongoService.java) gives access to a configured MongoClient, which is supposed to be shared by multiple services. The _updated_ method is invoked when configuration is found for this component. In the example code, the configuration is provided as a property file in the _run/conf_ directory. In the integration test however, the configuration is provided from code. 

{% highlight java %}
package chat.vertx.mongo.connection;

import java.util.Dictionary;

import org.apache.felix.dm.annotation.api.Component;
import org.apache.felix.dm.annotation.api.ConfigurationDependency;
import org.apache.felix.dm.annotation.api.ServiceDependency;
import org.apache.felix.dm.annotation.api.Start;
import org.apache.felix.dm.annotation.api.Stop;

import chat.vertx.mongo.MongoService;
import io.vertx.core.json.JsonObject;
import io.vertx.rxjava.core.Vertx;
import io.vertx.rxjava.ext.mongo.MongoClient;

@Component
public class VertxMongoService implements MongoService {

	@ServiceDependency
	private volatile Vertx vertx;
	
	private volatile JsonObject mongoconfig;
	private volatile MongoClient client;
	
	@Start
	public void start() {
		connect();
	}
	
	@Stop
	public void stop() {
		client.close();
		client = null;
	}

	@Override
	public MongoClient getClient() {
		return client;
	}
	
	@ConfigurationDependency(pid="vertx.mongo")
	public void updated(Dictionary<String, Object> properties) {
		if(properties != null ) {
			
			 mongoconfig = new JsonObject()
				        .put("connection_string", properties.get("url"))
				        .put("db_name", properties.get("db"));		
			connect();
		}
	}

	private void connect() {
		if(vertx != null) {
			client = MongoClient.createShared(vertx, mongoconfig);
		}
	}
	
}
{% endhighlight %}

Functional code that needs to work with Mongo simply injects the MongoService, and doesn't have to deal with configuration.

{% highlight java %}
@Component
public class MongoMessageStorage implements MessageStore {
	@ServiceDependency
	private volatile Vertx vertx;
	
	@ServiceDependency
	private volatile MongoService mongoService;

{% endhighlight %}


Integration testing 
-----

With OSGi we can very easily set up integration tests. These test look like plain junit tests, but actually run in a real OSGi framework together with the code under test. To run the tests we need to define a runtime; a list of bundles to install in the framework, in the test project's bnd.bnd file. The example is a test for the Mongo component that stores chat messages upon receiving them on the Eventbus. For this test to run we need to define a list of bundles that allows us to run Vertx and Mongo components, but we don't need to start a http server etc. Take a look at the _runbundles_ configuration in the [bnd.bnd](https://github.com/paulbakker/osgi-vertx-demo/blob/master/chat.vertx.messaging.storage.itest/bnd.bnd) file for an example. The test code itself is a mix of junit, Vertx and RX Java code.

{% highlight java %}
package chat.vertx.messaging.storage.test;

import static org.amdatu.testing.configurator.TestConfigurator.cleanUp;
import static org.amdatu.testing.configurator.TestConfigurator.configure;
import static org.amdatu.testing.configurator.TestConfigurator.createConfiguration;
import static org.amdatu.testing.configurator.TestConfigurator.createServiceDependency;

import java.util.Arrays;
import java.util.List;
import java.util.concurrent.TimeUnit;

import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import chat.api.ChatMessage;
import chat.vertx.messaging.storage.MessageStore;
import rx.Observable;
import rx.observers.TestSubscriber;
import io.vertx.rxjava.core.*;
import io.vertx.core.json.*;

public class MessageStoreTest {

	private volatile MessageStore serviceToTest;
	private volatile Vertx vertx;

	@Before
	public void setup() {
		configure(this)
			.add(createServiceDependency().setService(Vertx.class).setRequired(true))	
			.add(createConfiguration("vertx.mongo").set("url", "mongodb://localhost:27017").set("db", "vertxchat"))
			.add(createServiceDependency().setService(MessageStore.class).setRequired(true)).apply();
		
		serviceToTest.clear().toBlocking().subscribe();
	}
	
	@Test
	public void test() throws InterruptedException {
		
		ChatMessage msg1 = new ChatMessage(1, "User 1", "Hello");
		ChatMessage msg2 = new ChatMessage(2, "User 2", "Hello to you too!");
		ChatMessage msg3 = new ChatMessage(3, "User 1", "Chatting is fun");
		List<ChatMessage> msgList = Arrays.asList(msg1, msg2, msg3);
		Observable.from(msgList)
			.map(m -> Json.encode(m))
			.subscribe(m -> vertx.eventBus().publish("chat", m));
		TimeUnit.SECONDS.sleep(1);
		
		TestSubscriber<ChatMessage> testSubscriber = new TestSubscriber<>();
		serviceToTest.listMessages().flatMap(m -> Observable.from(m)).subscribe(testSubscriber);

		testSubscriber.awaitTerminalEvent();
		testSubscriber.assertNoErrors();
		testSubscriber.assertCompleted();
		testSubscriber.assertValueCount(3);
	}

	@After
	public void after() {
		cleanUp(this);
	}
}
{% endhighlight %}

Re-usablility of the code
-----

The heavy lifting to make Vertx play nicely with OSGi is done in the chat.vertx.bootstrap project. This setup is re-usable, the same approach can easily be used in other projects. This can even be done by just including the JAR file generated by this project, the source code wouldn't be necessary. I'm considering making this available as a library, but would like to first try it in a few more places to learn if anything else is missing. Feedback about this topic would be appreciated!

Conclusion
-----

Vertx is a really nice library, and works very well together with OSGi. Is Vertx better together with OSGi? Yes, I believe so. Although there are of course always other ways to approach things, OSGi services are an excellent programming model to achieve modular code bases, and the never-restart-the-app, and never-wait-for-a-build programming model is just too nice to ignore.

The code of this example is available on [GitHub](https://github.com/paulbakker/osgi-vertx-demo).