---
layout: post
title:  "What is a service mesh and why do I need one?"
date:   2021-01-17
category: service-mesh
tags: microservices service-mesh
---

With more and more organisations moving to microservice architectures, more and
more solutions are emerging to deal with the problems posed by distrubuted 
systems; one of which is the service mesh.

In this post we'll cover what a service mesh is, how it works, and why you might
want to use one.

## What is a Service Mesh?
A service mesh consists of some extra infrastructure you deploy alongside your
microservices to facilitate microservice-to-microservice communication. They 
generally consist of one or more "control plane" containers, along with a
"sidecar" container deployed alongside each of your microservice containers.
The sidecars act as proxies for the microservice containers, handling both 
incoming and outgoing network traffic. They communicate with the control plane
to enable things like service discovery, metrics reporting, and authentication;
and communicate with one-another to enable microservice-to-microservice 
communication.

## Example
Consider we have 2 microservices; a shopping cart service and a payments 
service. The shopping cart service wants to call and endpoint on the 
payments service to take a payment from a customer.

The shopping cart service will make a REST call to a URL that looks something
like `http://payments/make-payment`. This will be intercepted by its sidecar
proxy, which will in turn query the control plane for an IP address 
corresponding to the payments service. The control plane will provide the IP
address of the payments sidecar, so the shopping cart sidecar can call it
directly. The payments sidecar will act as a reverse-proxy, forwarding
traffic on to the payments container.

![Service Mesh Diagram](/assets/service-mesh/ServiceMesh.png)

The application traffic being proxied by the sidecars is often referred to as
the "data plane" of the service mesh, opposed to the control plane that supports
the operation of the mesh.

## Why do I need a service mesh?
Different service mesh implementations offer different functionality, but
we will take a look at some of the more common features offered by most
service mesh frameworks.

### Transparent Mutual TLS
Service meshes offer a simple way to implement mutual TLS for a zero-trust
architecture, without even having to modify your application code.

One of the services deployed as part of the control plane will be a
certificate authority. When a sidecar container first starts up, it will send
a certificate signing request (CSR) to the certificate authority, which will
respond with a signed certificate. The sidecars then use these certificates
for mutual TLS authentication with one-another when proxying data plane
traffic.

The application container and sidecar container are deployed as part of the same
pod, so can commuicate without traffic ever leaving the host server, meaning 
it's safe to use plain HTTP.

### Monitoring and Tracing
Sidecar proxies operate at layer 7 of the OSI model, meaning they are aware of
protocol-specific details such as HTTP and/or gRPC status codes, headers, etc.
This allows them to report all sorts of useful metrics back to the control 
plane, such as:
- Success/error rates
- Request timings
- Network latencies
- Request sources

Not only are these metrics aggregated for you to view (which can massively help
with debugging any issues), but they can also be used to influence the behaviour
of the mesh, e.g. enabling failover functionality (more on this in a minute).

Some service meshes also provide integration with tracing frameworks such as
[Jaeger] and [Zipkin].

[Jaeger]: https://www.jaegertracing.io/
[Zipkin]: https://zipkin.io/

### Load Balancing
If you're running multiple replicas of a container, you'll want a way to 
ensure traffic is distributed evenly between them. This is possible in a service
mesh since services only reference each other by name.

Continuing the example above, consider the case where we are running multiple
replicas of the payments service (each with their own sidecar). When the 
shoppning cart sidecar queries the control plane to retrieve an IP address for 
the payments service, the control plane can decide which replica's IP address to
return. Since the control plane is aware of all traffic flowing through the
mesh, it can ensure requests are distributed between replicas in whichever 
manner you require.

This functionality, when combined with metrics reporting, make service meshes
great tools for enabling canary deployments. You can deploy a new version of
a microservice alongside the current version, choose to route a small percentage
of traffic to the new version, and monitor for any errors. If everything looks
good then you can scale up the new version and kill off the older verison.

Similarly, they can easily support failover routing. If a replica of a service
starts responding with too many error codes, the control plane can stop routing 
requests to that replica. It may also be able to terminate the faulty replica 
and spin up a new one.

### Retries, Timeouts, and Circuit Breakers
The sidecar containers can implement request retries with a specified
backoff mechanism, custom request timeouts, and circuit breakers, all without
having to modify your application code. These can all be dynamically configured 
by the control plane on a per-service basis.

### Fault Injection
Chaos engineering, popularised by Netflix, is the practice of purposefully 
introducing failures into your system to see how well it copes. The idea is that
if your system can withstand these "simulated" errors, it has a better chance of
coping when things really go wrong.

To support this, sidecar containers can be configured to randomly simulate 
requests as failing.

## Summary
To summarise, a service mesh is an extra layer of infrastructure deployed
alongside your microservices to facilitate communication between them. They
introduce all sorts of useful functionality, all of which can be utilised 
without having to make any changes to your application code.

If you now want to get your hands on your very own service mesh, some commonly
used frameworks are:
- [Istio](https://istio.io/)
- [Linkerd](https://linkerd.io/)
- [HashiCorp Consul](https://www.consul.io/use-cases/multi-platform-service-mesh)
- [Traefik Mesh](https://traefik.io/traefik-mesh/)
