---
layout: post
title:  "Load balancing Kubernetes"
date:   2015-05-21 19:08
categories: Kubernetes
---

[Kubernetes](http://kubernetes.io) is excellent for running (web) applications in a clustered way. This blog will go into making applications deployed on Kubernetes available on an external, load balanced, IP address. Before diving into HTTP load balancers there are two Kubernetes concepts to understand: [Pods](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md) and [Replication Controllers](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/replication-controller.md). 

A Replication Controller describes a deployment of a container. One of the aspects to configure on a Replication Controller is the number of replicas. Each replica will run on a different Kubernetes node. This way an application or application component can be replicated on the cluster, which offers failover and load balancing over multiple machines. Each instance of a replica is called a Pod. Kubernetes will monitor Pods and will try to keep the number of Pods equal to the configured number of replicas. 

Load balancing Kubernetes apps
==

When we have multiple Pods running we still need a way to make the application available to the outside world, listening to a dns name like "http://myapp.io". Each Pod gets it's own virtual IP address (this works great combined with [Weave](https://github.com/weaveworks/weave)), but these IP addresses aren't useful to the outside world. 

Kubernetes has another concept, [Services](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md), that look useful in this context at first hand. A service is a proxy on top of Pods. This makes the service available on a more or less fixed IP address within Kubernetes, and it's clients don't know by which Pod a request is handled. Sounds like load balancing! Unfortunatly, the mechanism was designed to be used for components _within_ the Kubernetes cluster (think Micro Services) to communicate. It is possible to assign your own IP address to a service as well, which makes it possible to point your DNS to it, but we still miss features like support for SSL offloading, HTTP cache settings etc. Services are a very useful concept, but simply not designed for external HTTP load balancing. We need something much better.


Dynamic backends
==

The much better option is to use a load balancer like HA-Proxy, Nginx or Vulcan. The load balancer listens to HTTP(S) traffic, and forwards requests to Pods. This is nothing different than [configuring a proxy](http://nginx.com/resources/admin-guide/reverse-proxy/) in front of your standard Java/whatever application. This is possible because Pods are acceccible within the Kubernetes network on their own IP and the ports the container exposes. Make sure that the load balancer machine is part of the same virtual network as the Kubernetes Pods however!

There is still a problem. Pods are dynamic; they can come and go because of scaling or failover events, or because of deployment of a new application version. Every time a new Pod is started, it gets a new IP address. In a statically configured load balancer setup we would have to manually change the config files each time this happen, which is not acceptable in an environment like this.

The [Kubernetes API](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/api.md) can help us out here. You can create a watch on Pods with a given filter, and (de)register load balancer backends based on events from the watch. The final question is: How do we actually reconfigure the load balancer? For Nginx or Ha-Proxy we could use a template engine such as [confd](https://github.com/kelseyhightower/confd) to re-generate configuration files, and reload the configuration. This works, but is a bit cumbersome. We can do better.

Enter Vulcan
==

[Vulcan Proxy](http://vulcanproxy.com/) is a quite new load balancer specially created for these kind of dynamic environments. All it's configuration comes from etcd, which is  convenient because we need etcd for Kubernetes anyway. To register a new backend (our newly started Pod), we only have to add a new entry to etcd. Doing so is trivial, it can be done with a simple HTTP PUT or using one of the client libraries. 

For our Amdatu RTI environment, our commercial infrastructure offering, we created and open sourced a small tool that does exactly this; the Amdatu Vulcand registrator. To leverage the Kubernetes APIs the tool is writtin in Go, and is usually started as a Docker container. Let's configure an example where Kubernetes events trigger Vulcan configuration. In a basic setup, Vulcan needs three things: 

* A backend definition
* A frontend definition
* Backend servers

The first two will be static configuration in our example, but the backend servers are dynamically configured based on our Pods. The following etcd commands create the backend and frontend definitions.

{% highlight bash %}
etcdctl set /vulcand/backends/example/backend '{"Type": "http"}'

etcdctl set /vulcand/frontends/example/frontend '{"Type": "http", "BackendId": "example", "Route": "Path(`/`)"}'
{% endhighlight %}

If we would want to add a backend server manually (e.g. register a Pod manually) we could use the following command. Remember, we want to automate this part! In this case the Pod listens on IP address 10.200.1.2 and port 8080.

{% highlight bash %}
etcdctl set /vulcan/backends/example/servers/10.200.1.2 '{"URL": "http://10.200.1.2:8080"}'
{% endhighlight %}

Amdatu Vulcanized
==

The Amdatu Vulcanized registrator is available on [BitBucket](https://bitbucket.org/amdatulabs/amdatu-vulcanized), and as a [Docker container](https://registry.hub.docker.com/u/amdatu/amdatu-vulcanized) in the Docker Hub. It can be started as a Docker container:

{% highlight bash %}
docker run -d amdatu/amdatu-vulcanized app -pods "ws://[kubernetes-server]/api/v1beta3/namespaces/default/pods?watch=true" -etcd "[etcd-address]"
{% endhighlight %}

It will start watching Kubernetes for new or deleted Pods, and take care of (de)registration in etcd for Vulcan. With this, we have a fully dynamic load balancer configuration!

