---
title: kamon | Akka Http | Documentation
layout: documentation
---

Akka Http Integration
=====================

The `kamon-akka-http` module brings bytecode instrumentation for automatic traces and segments management and
automatic trace token propagation on your behalf. The details on what those features are about can be found in
the [base functionality] section. Here we will dig into the specific aspects of bringing support for them when using Akka Http.

<p class="alert alert-info">
The <b>kamon-akka-http</b> module requires you to start your application using the AspectJ Weaver Agent. Kamon will warn you
at startup if you failed to do so.
</p>

Server Side Tools
-----------------

For the Server-Side, the `kamon-akka-http` module brings instrumentation that will automatically start and finalize a trace automatically
for each request issued and produce the following metrics:

* Within category **akka-http-server**:
    * Number of open connections.
    * Number of active requests, recorded by 'status'_'trace name' as by 'trace name' only.
* Within category **trace**:
    * Elapsed time. Of course, this category is the same for any module.

### Providing a Name Generator ###

As you might know, we need to provide a suitable name for the traces, since it will be used for metric tracking. As with Spray and Play!
you should provide a **Name Generator Class** in order to name the traces, otherwise the default implementation names all traces as
UnnamedTrace. The Name Generator must extend the trait `kamon.akka.http.NameGenerator` and you have to indicate it via the configuration
key `kamon.akka-http.name-generator`.

Client-Side Tools
-----------------

For the client side, the bytecode instrumentation provided by the `kamon-akka-http` module hooks into Akka Http’s internals to automatically
start and finish segments for requests that are issued within a trace.

As with Spray Project, to consume HTTP services with Akka Http you can choose from three different levels of abstraction:
* Connection-Level Client-Side API
* Host-Level Client-Side API
* Request-Level Client-Side API

Currently this module supports only the Request-Level API, so if you use an other level, the segment measurement won't be performed.

For the Request-Level API the `kamon-akka-http` module measures the time during which the user application code is waiting for a request
issued to complete. To achieve it, Kamon attaches a callback to the `Future[HttpResponse]` returned by `HttpExt.singleRequest(..)`.
Finally, the following metrics are generated:

* Within category **trace-segment**:
    * Elapsed time, as with segment support on Spray and Play!
    
Trace Token propagation support
-----------------

The `kamon-akka-http` module automatically propagates the Trace Token both on the client and server side. [More info](http://kamon.io/integrations/web-and-http-toolkits/base-functionality/#automatically-propagate-the-trace-token).

[base functionality]: /integrations/web-and-http-toolkits/base-functionality/
