---
layout: post
title: 'Getting started with Kamon Akka Http'
date: 2016-08-19
categories: teamblog
---

Dear community, we are pleased to announce that after a long long wait, the first release of Kamon Akka Http is available.

This module is marked as experimental, which means that is in early access mode. We hope to can improve it based on your
suggestions and feedback, so feel free to send any comments or question via gitter, list mail or github.

As you can guess, the features introduced on this module are the same as the Kamon Spray module, so you will find a direct
relation between both modules.

Although the readme on github is clear enough to start to use this module, below we show a simple example to setup up a
basic monitoring with Kamon for a simple Akka Http project.


### Setting up the Project ###

We need to include in our `Build.scala` some dependencies:

{% code_block scala %}

val dependencies = Seq(
    "com.typesafe.akka"       %% "akka-http-experimental"       % "2.4.8",
    "io.kamon"                %% "kamon-akka-http-experimental" % "0.6.2",
    "io.kamon"                %% "kamon-log-reporter"   	    % "0.6.2" // (1)
    // ... and so on with all the others dependencies you need.
  )

val aspectJSettings = SbtAspectj.aspectjSettings ++ Seq(
    fork in run := true,
    connectInput in run := true,
    javaOptions in run <++= weaverOptions in Aspectj
  )
  
val main = Project(appName, file(".")).settings(libraryDependencies ++= dependencies)
              .settings(defaultSettings: _*)
              .settings(aspectJSettings) //(2)
{% endcode_block %}

1. We select [kamon-log-reporter] to test Kamon on the project.
2. The `kamon-akka-http` requires we to start the application using the AspectJ Weaver agent, so we need to add the
`-javaagent` JVM startup parameter pointing to the weaver’s file location. A simple way to achieve it is to use
[SBT AspectJ Plugin]

Additionally, depending on which metrics we want to measure, we must provide the necessary configuration.
For this example, we simply record all traces that is created and also indicate a provided Name Generator:

{% code_block typesafeconfig %}
kamon {

  akka-http.name-generator = playground.AkkaHttpNameGenerator

  metric {
    filters {
      trace.includes = [ "**" ]
    }
  }
}
{% endcode_block %}

### Create a simple WebServer ###

Let's start by creating a simple WebServer object that initializes the Akka Http Server with Kamon and records some metrics.

{% code_block scala %}

object WebServer extends App {

  Kamon.start()

  val (interface: String, port: Int) = ("0.0.0.0", 8080)

  implicit val system = ActorSystem()
  implicit val executor = system.dispatcher
  implicit val materializer = ActorMaterializer()

  val logger = Logging(system, getClass)

  val routes = {
    get {
      path("ok") {
        complete {
          "ok"
        }
      } ~
        path("go-to-outside") {
          complete {
            Http().singleRequest(HttpRequest(uri = "http://ip-api.com:80/"))
          }
        } ~
        path("internal-error") {
          complete(HttpResponse(InternalServerError))
        } ~
        path("fail-with-exception") {
          throw new RuntimeException("Failed!")
        }
    }
  }

  val bindingFuture = Http().bindAndHandle(routes, interface, port)

  bindingFuture.map { serverBinding ⇒

    logger.info(s"Server online at http://$interface:$port/\nPress RETURN to stop...")

    StdIn.readLine()

    logger.info(s"Server is shutting down.")

    serverBinding
      .unbind() // trigger unbinding from the port
      .flatMap(_ ⇒ {
        Kamon.shutdown()
        system.terminate()
      }) // and shutdown when done
  }

}

{% endcode_block %}

Here is the imports that you need to include to run this code:

{% code_block scala %}

import akka.actor.ActorSystem
import akka.event.Logging
import akka.http.scaladsl._
import akka.http.scaladsl.model.StatusCodes._
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.Directives._
import akka.stream.ActorMaterializer
import kamon.Kamon

import scala.io.StdIn

{% endcode_block %}


### Select a Kamon Backend ###

This time will we use the [kamon-log-reporter]. This module is not meant to be used in production environments, but it
certainly is a convenient way to test Kamon and know in a quick manner what's going on with our application in
development, moreover like all Kamon modules it will be picked up from the classpath and started at runtime, just by adding the
dependency to our classpath we are good to go.

Additionally we can find more info about how to configure [modules] and supported backends ([StatsD], [Datadog], [New
Relic] and [Your Own]) in the docs.

### Start the Server ###

We can run this application directly by typing `sbt run` from the console.

The output we will see should be something like this if we hit some of the endpoints we’ve set up.

* **curl** *http://localhost:8080/ok*
* **curl** *http://localhost:8080/go-to-outside*
* **curl** *http://localhost:8080/internal-error*
* **curl** *http://localhost:8080/fail-with-exception*


{% code_block text %}
+--------------------------------------------------------------------------------------------------+
|                                                                                                  |
|    Trace: /go-to-outside                                                                         |
|    Count: 5                                                                                      |
|                                                                                                  |
|  Elapsed Time (nanoseconds):                                                                     |
|    Min: 186646528    50th Perc: 186646528      90th Perc: 396361728      95th Perc: 396361728    |
|                      99th Perc: 396361728    99.9th Perc: 396361728            Max: 396361728    |
|                                                                                                  |
+--------------------------------------------------------------------------------------------------+
{% endcode_block %}

{% code_block text %}
+--------------------------------------------------------------------------------------------------+
|                                                                                                  |
|    Trace: /internal-error                                                                        |
|    Count: 3                                                                                      |
|                                                                                                  |
|  Elapsed Time (nanoseconds):                                                                     |
|    Min: 663552       50th Perc: 1900544        90th Perc: 1933312        95th Perc: 1933312      |
|                      99th Perc: 1933312      99.9th Perc: 1933312              Max: 1933312      |
|                                                                                                  |
+--------------------------------------------------------------------------------------------------+
{% endcode_block %}

{% code_block text %}
+--------------------------------------------------------------------------------------------------+
|                                                                                                  |
|    Trace: /ok                                                                                    |
|    Count: 2                                                                                      |
|                                                                                                  |
|  Elapsed Time (nanoseconds):                                                                     |
|    Min: 626688       50th Perc: 626688         90th Perc: 16908288       95th Perc: 16908288     |
|                      99th Perc: 16908288     99.9th Perc: 16908288             Max: 16908288     |
|                                                                                                  |
+--------------------------------------------------------------------------------------------------+
{% endcode_block %}

### Enjoy! ###

There it is, all your metrics data available to be sent into whatever tool you like. From here, you should be able to
instrument your applications as needed.

You can check out the full source code of [Akka Http Kamon Example] used in this tutorial.

[Akka Http Kamon Example]: https://github.com/kamon-io/kamon-akka-http/tree/v0.6.2/kamon-akka-http-playground/src/main
[modules]: /core/modules/using-modules/
[kamon-log-reporter]: /backends/log-reporter/

[StatsD]: /backends/statsd/
[Datadog]: /backends/datadog/
[New Relic]: /backends/newrelic/
[Your Own]: /core/metrics/subscription-protocol/
[SBT AspectJ Plugin]: https://github.com/sbt/sbt-aspectj
