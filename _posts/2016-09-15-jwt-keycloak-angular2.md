---
layout: post
title:  "JWT authentication with Vert.x, Keycloak and Angular 2"
date:   2016-09-15 22:15:19
categories: java
comments: true
---

Almost every web app requires some kind of user management, authentication and authorization. 
This is often custom build.
I can't even count the number of times I created something like this as part of a project.
It's also non trivial to create something truly reusable for this.
With new standards emerging like Openid Connect and JWT, things start to look more promising.

For the development of [Cloud RTI](https://www.cloud-rti.com/) I started searching for something that would support the following:

* Authentication and authorization in the frontend (Angular 2)
* Secure REST endpoints
* User and role management
* Integration with social providers (Facebook, Google etc.)
* Single sign-on over several applications

There are several available tools and hosted services available that satisfy these requirements.
The tool that we ended up choosing is [Keycloak](http://www.keycloak.org/), a project from Red Hat.
Keycloak is open source and is very actively developed.
It supports several standards such as Openid Connect, JWT and OAuth2.

The example application is structured the same way as many real applications.
It has an Angular 2 frontend, and users must login to use the applications.
Users can login using an account they create for the application, or using a social account.
The frontend uses a Vert.x REST backend.
Calls to the backend can only be done by authenticated users, and the backend logs which user requests the API (for demo purposes).
The demo application uses Keycloak to handle authentication, authorization and user managment.
The example project is available on [GitHub](https://github.com/paulbakker/vertx-angular2-keycloak-demo).

Getting Keycloak to run
--

Before we can run the demo project we need to start Keycloak and set it up.
The easiest way to get started with Keycloak is by using the available Docker container.
When taking Keycloak to production you should probably think about persistence a little better, but we don't need that right now.
Start Keycloak using the following command:

{% highlight bash %}
docker run -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -p 8080:8080 jboss/keycloak
{% endhighlight %}

You can now open `localhost:8080` and login to Keycloak user admin/admin.
In Keycloak we can setup different _reamls_.
A realm contains users and groups.
By creating multiple realms we can use the same Keycloak installation for several applications.
In this case we'll just use the master realm however.
Before we can integrate the demo application we need to configure Keycloak for this.
This configuration is done in a _client_.
Create a new client with Client ID `demo` and Client Protocol `openid-connect`.
In the screen that follows we can setup the client for a specific authentication flow.
We want a setup where the user is redirected to Keycloak for authentication when he or she opens the application.
After successful authentication the user should be redirect back to the application.
We can do so in Keycloak using the `Standard flow`.
The only configuration left is setting the right urls for our application.
In this example we'll just run on localhost on port 8000.

{% highlight bash %}
Valid Redirect Urls: http://localhost:8000/*
Web Origins: http://localhost:8000
{% endhighlight %}

![Keycloak configuration](/images/posts/keycloak/keycloak_client.png)

The Vert.x backend
--

The Vert.x backend starts a HTTP server that does the following:

* Serve the frontend static files
* Return a list of books on /api/books
* Log the user requesting /api/books

Only authenticated users can access /api/books.

{% highlight java %}
public class Main {
     Vertx vertx = Vertx.vertx();

        Router router = Router.router(vertx);
        Handlers handlers = new Handlers();

        router.route("/api/*").handler(CorsHandler.create("http://localhost:4200/*")
                .allowedMethod(HttpMethod.GET)
                .allowedHeader("Authorization")
                .allowedHeader("Content-Type")
                .allowCredentials(true));

        JsonObject config = new JsonObject()
                .put("public-key", "KEYCLOAK_PUBLIC_KEY")
                .put("permissionsClaimKey", "realm_access/roles");
        JWTAuth authProvider = JWTAuth.create(vertx, config);

        router.route("/api/*").handler(JWTAuthHandler.create(authProvider));
        router.route("/api/*").handler(handlers::userLogger);


        router.route("/api/books").handler(handlers::listBooks);

        StaticHandler keycloakStaticHandler = StaticHandler.create("webroot/keycloak");
        router.route("/keycloak/*").handler(keycloakStaticHandler::handle);

        StaticHandler staticHandler = StaticHandler.create("webroot/dist");
        router.route("/*").handler(staticHandler::handle);


        vertx.createHttpServer().requestHandler(router::accept).listen(8000);
}
{% endhighlight %}

Here we setup the handlers for the application.
If you're not familiar with Vert.x, you might want to learn about handlers first [here](http://vertx.io/docs/vertx-web/java/#_handling_requests_and_calling_the_next_handler).
The most interesting part is the setup of the auth provider.
We use JWT (JSON Web Token), which is supported out of the box by Vert.x.
A token in JWT is self contained. 
This means we can validate the token using the public key, and can get information about the user including the user's roles directly from the token.
No calls to Keycloak are required for this.
This is both convenient and efficient.
Off course this assumes that the user is already logged in, which we will take care of in the frontend.
The only missing part in this configuration is the actual public key, because this is different for every Keycloak installation.
In Keycloak, select _Realm settings / keys_ and copy the public key to the value that in the code above says `KEYCLOAK_PUBLIC_KEY`.

Note that instead of Keycloak we could use another JWT compatible provider with mostly the same configuration.

The actual reqeust handlers are listed below.
The `userLogger` shows how Vert.x makes it easy to get user information from the request, including the user's roles.
That's all!
The backend is ready to be used.

{% highlight java %}
public class Handlers {
    private final static Logger log = LoggerFactory.getLogger(Handlers.class);
    private final List<Book> books = new ArrayList<>();

    public Handlers() {
        books.add(new Book("Java 9 modularity", "java", "modularity", "jigsaw"));
        books.add(new Book("Building Cloud Apps with OSGi", "java", "modularity", "osgi"));
    }

    public void listBooks(RoutingContext rc) {
        rc.response().end(Json.encode(books));
    }

    public void userLogger(RoutingContext rc) {
        User user = rc.user();
        if (user != null) {
            JsonObject principal = user.principal();
            log.info("Request by {}", principal.getString("preferred_username"));
        } else {
            log.error("No logged in user");
        }

        rc.next();
    }
}
{% endhighlight %}

The backend is finished, now we only have to setup the frontend.

Setting up Angular 2 for JWT
--

The frontend for the demo application is created using Angular CLI and can be found in the `webroot` directory.
The project generated by Angular CLI is mostly unchanged, it's only modified to integrate with Keycloak and read some data from the backend.

In the frontend we have to take care of two things:

* Make sure the user is authenticated and redirect to the Keycloak login page otherwise
* Add the JWT token to each backend request

The first can be done by using the Keycloak JavaScript API.
The following code bootstraps the Angular application in `main.ts`.
Before doing so it sets up Keycloak, which will take care of redirecting to Keycloak when the user is not logged in.
Also, the JWT token is refreshed automatically, preventing the user to be logged out after a few minutes.
The Keycloak object is created with a parameter `keycloak/keycloak.json`.
This is the configuration file for our Keycloak client.
You can find this configuration in Keycloak under your client configuration.
Copy this configuration to `keycloak/keycloak.json` in the webroot directory.

![Keycloak client json](/images/posts/keycloak/keycloak_client_json.png)

Keycloak has a JavaScript API that we can install using npm.

{% highlight bash %}
npm install keycloak-js --save
{% endhighlight %}

Now that Keycloak is availble in the project we can use the API to setup the redirect.

{% highlight javascript %}
import './polyfills.ts';

import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { enableProdMode } from '@angular/core';
import { environment } from './environments/environment';
import { AppModule } from './app/';
import * as Keycloak from 'keycloak-js';

if (environment.production) {
  enableProdMode();
}

let keycloak = Keycloak('keycloak/keycloak.json');
window['_keycloak'] = keycloak;

window['_keycloak'].init(
    {onLoad: 'login-required'}
)
    .success(function (authenticated) {

      if (!authenticated) {
        window.location.reload();
      }

      // refresh login
      setInterval(function () {

        keycloak.updateToken(70).success(function (refreshed) {
          if (refreshed) {
            console.log('Token refreshed');
          } else {
            console.log('Token not refreshed, valid for '
                + Math.round(keycloak.tokenParsed.exp + keycloak.timeSkew - new Date().getTime() / 1000) + ' seconds');
          }
        }).error(function () {
          console.error('Failed to refresh token');
        });

      }, 60000);

      console.log("Loading...");

      platformBrowserDynamic().bootstrapModule(AppModule);

    });

{% endhighlight %}

When not logged in yet, users will be redirected to the Keycloak login page.
After login, they we be redirected back to the application.
The next step is to pass the JWT token to the backend for REST calls.
This way the backend can authenticate the user, and potentially inspect the user's roles to authorize a call as well.
The easiest way to do this is by using _angular2-jwt_, an Angular 2 component created by Auth0.
This basically extends the Angular 2 HTTP service with things like adding tokens to requests.
First install the component using npm.

{% highlight bash %}
npm install angular2-jwt --save
{% endhighlight %}

Now we can setup the provider in `app.module.ts`.

{% highlight javascript %}
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpModule } from '@angular/http';
import { provideAuth } from 'angular2-jwt';
import { AppComponent } from './app.component';
import { DemoService } from './demo.service'

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpModule
  ],
  providers: [DemoService,
    provideAuth({
      globalHeaders: [{'Content-Type': 'application/json'}],
      noJwtError: true,
      tokenGetter: () => {
        return window['_keycloak'].token;
      }
    })],
  bootstrap: [AppComponent]
})
export class AppModule { }
{% endhighlight %}

That's it!
The frontend is now also completed.
When not logged in, a user will now be redirected to Keycloak.
We can manage users and roles in Keycloak, and role information is part of the JWT token, which we can access both in Vert.x and the frontend.
Setting up additional authentication providers, such as Google and Facebook is also, as shown in the docs.
The example project is available on [GitHub](https://github.com/paulbakker/vertx-angular2-keycloak-demo).
