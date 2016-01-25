---
layout: post
title:  "Hosting static sites on Kubernetes"
date:   2016-01-10 22:15:19
categories: kubernetes
comments: true
---

What if you have a completely static website (yup, they still exist) or a JavaScript web frontend and want to host it on Kubernetes? This post describes two option, one for projects that do not require a build, and one for projects that do.

Hosting a fully static site
------

If no build is required for the project (pure HTML/CSS/JavaScript), it is not necessary to create a Docker image for the project. Instead we use two standard containers:

* Nginx, using the default image
* [git-sync](https://github.com/kubernetes/contrib/tree/master/git-sync), a tool to keep in sync with a git repository.

The git-sync tool pulls the resources from a git repository, and makes them available to Nginx, which takes care of making them available over HTTP. This is an example of a Pod with multiple containers; Nginx being the main container and git-sync a *side car* that supports the main container.

Using this approach we don't need to build any Docker container, and we don't need to re-deploy to Kubernetes upon changes. The git-sync container comes from te Kubernetes contrib project, but contains two issues ([397](https://github.com/kubernetes/contrib/pull/397) and [398](https://github.com/kubernetes/contrib/pull/398)) that I fixed. A Docker image containing these fixes is available as [paulbakker/git-sync](https://hub.docker.com/r/paulbakker/git-sync).

The git-sync and Nginx container must be able to share files. For this we can use a Kubernetes *volume*. Both containers mount the same volume, that way Nginx can serve files pulled by the git-sync tool. An example Pod definition in JSON looks as follows:

{% highlight java %}
"podspec": {
  "containers": [{
    "image": "nginx",
    "name": "website",
    "ports": [{
      "containerPort": 80
    }],

    "volumeMounts": [{
      "mountPath": "/usr/share/nginx/html",
      "name": "www",
      "readOnly": true
    }]
  }, {
    "image": "paulbakker/git-sync",
    "name": "git-sync",
    "imagePullPolicy": "Always",
    "env": [{
      "name": "GIT_SYNC_REPO",
      "value": "https://github.com/somerepo/somerepo.git"
    }, {
      "name": "GIT_SYNC_WAIT",
      "value": "10"
    }],
    "volumeMounts": [{
      "mountPath": "/git",
      "name": "www"
    }]
  }],
  "volumes": [{
    "name": "www",
    "emptyDir": {}
  }]
}
{% endhighlight %}

When the git repo is updated, the hosted site on Kubernetes will be automatically updated as well.

Hosting a UI that requires a build
-------

Most UI projects that we work on are not pure HTML/JavaScript. More often than not we use TypeScript, we need minification, or other things that require a build step to generate the files that are actually served to the browser.

The git-sync approach described above doesn't really work in that case. It's better to create a Docker image that contains the generated files from a build server. It turns out that building projects with all kind of Node modules reliably can be a bit of a challange on different environments. I found it to be easier to make the build part of the actual Docker image, so that it becomes more reproducable. There is already a standard Node Docker image that does this.

For the example, let's assume our project has a [project.json](https://docs.npmjs.com/files/package.json) file.

{% highlight javascript %}
{
  "name": "demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "tsc": "tsc"    
  },
  "author": "",
  "license": "ISC",
  "devDependencies": {  
    "typescript": "^1.7.3"
  }
}
{% endhighlight %}

To build the project we need to invoke the 'tsc' command. We can use the following Docker file to do so. 

{% highlight bash %}
FROM node:argon-wheezy

RUN mkdir -p /www
COPY package.json /www
WORKDIR /www
RUN npm install

COPY . /www
RUN npm run tsc

CMD ["tail", "-f", "/dev/null"]
{% endhighlight %}

Of course the build could be more complicated, either using more npm scripts, or using something like Grunt. The Docker file above creates a container that *just* contains our generated files, it doesn't actually contain a web server. We will use the same side-car approach to hook up these files to an Nginx container. 
Kubernetes doesn't like containers that exit. That means we need to do something to keep the container alive, which explains the *tail -f* at the end of the Dockerfile.

Now let's look at the Pod definition for this setup.

{% highlight bash %}
"podspec": {
  "containers": [{
      "image": "nginx",
      "name": "nginx",
      "ports": [{
        "containerPort": 80
      }],
      "volumeMounts": [{
        "mountPath": "/usr/share/nginx/html",
        "name": "www",
        "readOnly": true
      }]
    },

    {
      "imagePullPolicy": "Always",
      "image": "paulbakker/example",
      "name": "example-frontend",
      "volumeMounts": [{
        "mountPath": "/data",
        "name": "www"
      }],
      "lifecycle": {
        "postStart": {
          "exec": {
            "command": ["cp", "-R", "/www/.", "/data"]
          }
        }
      }
    }
  ],

  "volumes": [{
    "name": "www",
    "emptyDir": {}
  }]
}
{% endhighlight %}

Again we're using a standard Nginx image. The frontend image uses a *lifecycle* hook to copy the generated files to the shared volume. This is necessary because mounting an existing directory as a volume will empty it!

As an alternative you could make Nginx part of your own Docker image. That way we don't need the side car approach, but the Dockerfile becomes a bit more complicated. Both approaches work well, it's a matter of preference which approach to use.

{% highlight bash %}
FROM node:argon-wheezy

RUN apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
RUN echo "deb http://nginx.org/packages/mainline/debian/ wheezy nginx" >> /etc/apt/sources.list

ENV NGINX_VERSION 1.9.9-1~wheezy

RUN apt-get update && \
    apt-get install -y ca-certificates nginx=${NGINX_VERSION} && \
    rm -rf /var/lib/apt/lists/*

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80

RUN mkdir -p /usr/share/nginx/html
COPY package.json /usr/share/nginx/html/
WORKDIR /usr/share/nginx/html/
RUN npm install

COPY . /usr/share/nginx/html/
RUN npm run tsc

CMD ["nginx", "-g", "daemon off;"]
{% endhighlight %}
