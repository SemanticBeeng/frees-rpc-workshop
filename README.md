[![Build Status](https://travis-ci.org/frees-io/frees-rpc-workshop.svg?branch=master)](https://travis-ci.org/frees-io/frees-rpc-workshop)

# frees-workshop: Building Purely Functional Microservices

In this workshop, you will learn how to build from a purely functional application from scratch and expose it as a microservice with Freestyle and Freestyle RPC.

This will be a hands-on coding session where we will architect a small application based on Algebras and Modules that can be exposed as an RPC microservice supporting Protobuf and Avro serialization protocols.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Basic Freestyle Structure](#basic-freestyle-structure)
- [App Domain and Data Generator](#app-domain-and-data-generator)
- [Freestyle RPC - Protocols and Services](#freestyle-rpc---protocols-and-services)
  - [Protocol Definition](#protocol-definition)
  - [Service Implementation](#service-implementation)
- [RPC Server and RPC Client usage](#rpc-server-and-rpc-client-usage)
  - [RPC Server](#rpc-server)
  - [App Client Demo](#app-client-demo)
- [Server Streaming](#server-streaming)
- [Client Streaming](#client-streaming)
- [What's Next](#whats-next)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Before starting, just a mention that you can "roll" – view and navigate – the Git commit history thanks to https://github.com/sbt/sbt-groll, and see/compile/test step by step all the content in this repository. Basic usage:

* See the full commit history:

```bash
sbt:functional-microservices> groll list
[info] == a00df1c 6 - RPC Client Streaming
[info]    4294dda 5 - RPC Server Streaming
[info]    cbbc324 4 - RPC Server and RPC Client usage
[info]    1fd1a47 3 - RPC Unary Services - Protocols
[info]    8ba3f75 2 - App Domain and Data Generator
[info]    bee6895 1 - Basic Freestyle project structure
```

* Move to the first commit (`1 - Basic Freestyle project structure`):

```bash
groll move bee6895
```

* Move to the next commit:

```bash
groll next
```

* Moves to the previous commit:

```bash
groll prev
```

## Basic Freestyle Structure

We are going to use the [freestyle-seed](https://github.com/frees-io/freestyle-seed.g8) [giter8](https://github.com/foundweekends/giter8) template to create the basic project structure:

```bash
sbt new frees-io/freestyle-seed.g8
```

Result:

```bash
name [Project Name]: frees-rpc-workshop
projectDescription [Project Description]:  Freestyle at Scala eXchange
project [project-name]: functional-microservices
package [freestyle]: scalaexchange
freesVersion [0.4.6]:

Template applied in ./frees-rpc-workshop
```

Run the example:

```bash
cd frees-rpc-workshop
sbt run
```

## App Domain and Data Generator

In this section, we'll review the different models that we will be using during the workshop. We'll use the [RFM Analysis](https://en.wikipedia.org/wiki/RFM_(customer_value)) as domain problem. RFM is a method used for analyzing customer value. It is commonly used in marketing and has wide spread applications in retail industries. RFM stands for:

* Recency – How recently did the customer purchase?
* Frequency – How often do they purchase?
* Monetary Value – How much do they spend?

First off, let's add some new sbt settings (and sbt modules) to allow us to put our code in the right place.

Add the following to your project's `build.sbt` file:

* Common Settings and dependency we'll use for every module in our project:

```scala
lazy val commonSettings: Seq[Def.Setting[_]] = Seq(
  resolvers += Resolver.bintrayRepo("beyondthelines", "maven"),
  addCompilerPlugin("org.scalameta" % "paradise" % "3.0.0-M10" cross CrossVersion.full),
  libraryDependencies ++= Seq(
    "io.frees" %% "frees-core" % freesV,
    "io.frees" %% "frees-rpc" % "0.4.1",
    "org.scalameta" %% "scalameta" % "1.8.0"),
  scalacOptions += "-Xplugin-require:macroparadise",
  scalacOptions in(Compile, console) ~= (_ filterNot (_ contains "paradise")) // macroparadise plugin doesn't work in repl yet.
)
```

* Modify the current settings in our current `functional-microservices` sbt module, you can use the following configuration:

```scala
// Common module:
lazy val `functional-microservices` = project
  .in(file("."))
  .settings(name := "functional-microservices")
  .settings(moduleName := "functional-microservices")
  .settings(description := "Freestyle at Scala eXchange")
  .settings(commonSettings)
```

* Remove all the code provided by the giter8 template (you can just directly remove the `scalaexchange` folder/package), since we are building everything from scratch in the next steps. In fact, from now on, `functional-microservices` sbt module will be used as the common build, which will be visible for the rest of the sbt modules.

* Our model, that can be placed in the common space within our project ([./src/main/scala](./src/main/scala)):

  * [./src/main/scala/models.scala](./src/main/scala/models.scala):

```scala
package scalaexchange

sealed trait EventType
case object ProcessedCheckout   extends EventType
case object UnprocessedCheckout extends EventType
case object Login               extends EventType

case class UserEvent(userId: Int, eventType: EventType, date: String)
```

  * [./src/main/scala/errors.scala](./src/main/scala/errors.scala):

```scala
package scalaexchange

case class DataGenerationException(message: String, maybeCause: Option[Throwable] = None)
  extends RuntimeException(message) {

  maybeCause foreach initCause
}
```

* Now, let's define three new `sbt` modules, also in our `build.sbt`, that will be needed later on:

```scala
// Data Generator:
lazy val `data-generator` =
  project.in(file("data-generator"))
    .settings(moduleName := "data-generator")
    .settings(commonSettings)
    .aggregate(`functional-microservices`)
    .dependsOn(`functional-microservices`)
    .settings(
      libraryDependencies ++= Seq(
        "joda-time" % "joda-time" % "2.9.9",
        "io.monix" %% "monix" % "3.0.0-M2",
        "org.scalacheck" %% "scalacheck" % "1.13.4",
        "com.47deg" %% "scalacheck-toolbox-datetime" % "0.2.3"
      )
    )

// RPC definitions and implementations:
lazy val services = project
  .in(file("services"))
  .settings(moduleName := "rpc-services")
  .settings(commonSettings)
  .aggregate(`functional-microservices`)
  .dependsOn(`functional-microservices`)

// Our application where we will test everything we are building:
lazy val app = project
  .in(file("app"))
  .settings(moduleName := "app")
  .settings(commonSettings)
  .aggregate(`data-generator`, services)
  .dependsOn(`data-generator`, services)
```

* Streaming Data Generation: you can copy this code verbatim from this path [./data-generator/src/main/scala/](./data-generator/src/main/scala/) inside the `data-generator` sbt module folder, given that this code is out of the scope of this workshop. Notice that this should be under the `data-generator` folder.

* Checkpoint: let's test this streaming data generation before moving onto the next section. We create the `AppStreamingService` class inside the `app` sbt module folder([./app/src/main/scala/AppStreamingService.scala](./app/src/main/scala/AppStreamingService.scala)). Notice that this should be under the `app` folder.

```scala
package scalaexchange

import monix.execution.Scheduler

import scala.concurrent.Await
import scala.concurrent.duration.Duration

object AppStreamingService {

  implicit val S: Scheduler = monix.execution.Scheduler.Implicits.global

  def main(args: Array[String]): Unit = {
    val streamingService: StreamingService = new StreamingService

    Await.ready(streamingService.userEventsStream.completedL.runAsync, Duration.Inf)
  }

}
```

Now you can run this application:

```bash
sbt app/run
```

The output should be similar to:

```bash
[info] Running scalaexchange.AppStreamingService
* New Event 👍  --> UserEvent(76,ProcessedCheckout,2017-12-14T15:25:48.820Z)
* New Event 👍  --> UserEvent(21,UnprocessedCheckout,2017-12-05T05:08:20.558Z)
* New Event 👍  --> UserEvent(16,ProcessedCheckout,2017-12-07T11:35:20.559Z)
* New Event 👍  --> UserEvent(80,UnprocessedCheckout,2017-12-07T17:32:24.181Z)
* New Event 👍  --> UserEvent(28,Login,2017-12-08T14:50:48.704Z)
* New Event 👍  --> UserEvent(63,UnprocessedCheckout,2017-12-03T06:22:30.471Z)
* New Event 👍  --> UserEvent(94,ProcessedCheckout,2017-12-08T08:59:24.241Z)
* New Event 👍  --> UserEvent(74,Login,2017-12-12T08:03:18.996Z)
* New Event 👍  --> UserEvent(43,UnprocessedCheckout,2017-12-12T03:16:03.142Z)
* New Event 👍  --> UserEvent(32,ProcessedCheckout,2017-12-09T10:09:40.566Z)
* New Event 👍  --> UserEvent(74,UnprocessedCheckout,2017-12-05T20:39:46.105Z)
* New Event 👍  --> UserEvent(74,ProcessedCheckout,2017-12-07T03:36:50.931Z)

... // Feel free to stop it, it has no end.
```

## Freestyle RPC - Protocols and Services

Previously, we added the `frees-rpc` dependency to your project's `build.sbt` file:

```scala
libraryDependencies += "io.frees" %% "frees-rpc" % "0.4.1"
```

Hence, we are able to start with the definition and implementation of our microservices, using `frees-rpc`. Everything in this section will happen in the `services` sbt module.

First of all, we are going to move the common implicits to a common place, since we need to start thinking about the fact that we'll have several runnable parts, like server and client applications.

At this point, we only need to move the implicit `ExecutionContext` to our common space. Hence, pick it up from `scalaexchange.app.AppStreamingService` and create the following object to put it there (file path [./src/main/scala/implicits.scala](./src/main/scala/implicits.scala)):

```scala
package scalaexchange

import monix.execution.Scheduler

trait CommonImplicits {

  implicit val S: Scheduler = monix.execution.Scheduler.Implicits.global

}

object implicits extends CommonImplicits
```

Next, change the class `AppStreamingService` accordingly:

```scala
package scalaexchange
package app

import scala.concurrent.Await
import scala.concurrent.duration.Duration
import scalaexchange.datagenerator.StreamingService

object AppStreamingService extends CommonImplicits {

  def main(args: Array[String]): Unit = {
    val streamingService: StreamingService = new StreamingService

    Await.ready(streamingService.userEventsStream.completedL.runAsync, Duration.Inf)
  }

}
```

Let's focus now on the most important part of this section: creating a protocol definition with a service implementation, with the help of [Freestyle RPC](https://github.com/frees-io/freestyle-rpc).

### Protocol Definition

In this first approach, we'll define a simple unary service, receiving an empty request and returning a single response. In fact, we are defining a service which will return the set of segments that our `RFM` microservices will manage from now on, in order to clasify the different users.

* Segment definition:
  * title: String
  * minRecency: Int
  * maxRecency: Int
  * minFrequency: Int
  * maxFrequency: Int
  * minMonetary: Int
  * maxMonetary: Int
* Example showing the list of Segments that our service could return:
  * Segment("Champions", 4, 5, 4, 5, 4, 5)
  * Segment("Loyal Customers", 2, 5, 3, 5, 3, 5)
  * Segment("Potential Loyalist", 3, 5, 1, 3, 1, 3)
  * Segment("New Customers", 4, 5, 0, 1, 0, 1)
  * Segment("Customers Needing Attention", 3, 4, 0, 1, 0, 1)
  * Segment("About To Sleep", 2, 3, 0, 2, 0, 2)
  * Segment("Can't Lose Them", 0, 1, 4, 5, 4, 5)
  * Segment("At Risk", 0, 2, 2, 5, 2, 5)
  * Segment("Hibernating", 1, 2, 1, 2, 1, 2)
  * Segment("Lost", 0, 2, 0, 2, 0, 2)
* Protocol Definition, which we are going to define it at [./services/src/main/scala/protocol.scala](./services/src/main/scala/protocol.scala), as shown below:

```scala
package scalaexchange
package services

import freestyle.rpc.protocol._

object protocol {

  final case class Segment(
      title: String,
      minRecency: Int,
      maxRecency: Int,
      minFrequency: Int,
      maxFrequency: Int,
      minMonetary: Int,
      maxMonetary: Int
  )

  final case class SegmentList(list: List[Segment])

  @service
  trait RFMAnalysisService[F[_]] {

    @rpc(Avro) def segments(empty: Empty.type): F[SegmentList]

  }

}
```

Notice the `@service` and `@rpc(Avro)` annotations, both are provided by Freestyle RPC. The first one allows us to define an RPC service. The second one though, brings us the ability to define an RPC service. In this case, we are serialising with `Avro` but `Protocol Buffers` is also supported (in that case you should use `Protobuf` instead).

### Service Implementation

So far, so good. What about the implementation? Initially, let's provide a simple implementation to serve the user segment list (`SegmentList`). We could do this at [./services/src/main/scala/runtime/RFMAnalysisServiceHandler.scala](./services/src/main/scala/runtime/RFMAnalysisServiceHandler.scala):

```scala
package scalaexchange
package services
package runtime

import cats.Applicative
import freestyle.rpc.protocol._

import scalaexchange.services.protocol._

class RFMAnalysisServiceHandler[F[_]: Applicative] extends RFMAnalysisService[F] {

  private[this] val segmentList: List[Segment] = List(
    Segment("Champions", 4, 5, 4, 5, 4, 5),
    Segment("Loyal Customers", 2, 5, 3, 5, 3, 5),
    Segment("Potential Loyalist", 3, 5, 1, 3, 1, 3),
    Segment("New Customers", 4, 5, 0, 1, 0, 1),
    Segment("Customers Needing Attention", 3, 4, 0, 1, 0, 1),
    Segment("About To Sleep", 2, 3, 0, 2, 0, 2),
    Segment("Can't Lose Them", 0, 1, 4, 5, 4, 5),
    Segment("At Risk", 0, 2, 2, 5, 2, 5),
    Segment("Hibernating", 1, 2, 1, 2, 1, 2),
    Segment("Lost", 0, 2, 0, 2, 0, 2)
  )

  override def segments(empty: Empty.type): F[protocol.SegmentList] =
    Applicative[F].pure(SegmentList(segmentList))

}
```

As you can see, we are hardcoding the segment list. Nonetheless, this list could be fetched from a database, but, we are good for now with this dummy implementation.

As takeaways, with this protocol definition, we have two things:

* A service that can be attached to any RPC server.
* Auto-derived rpc client of this service.

In the next section, we are going to see both pieces in action, end-to-end.

## RPC Server and RPC Client usage

As we mentioned at the end of the last section, we are:

* Bootstrapping an RPC Server (on port `8080`, for example). Before doing that, we need to bind the previous service to this server.
* Invoke the service from the client side, using the auto-generated client for the previously defined protocol.

For both challenges, we need to apply effects to an specific concurrent Monad. We've chosen `cats.effect.IO`, from [cats-effect](https://github.com/typelevel/cats-effect), but it could be any other. For this reason, we are adding the sbt dependency to our [build.sbt](./build.sbt) file, which will be provided transitively by [frees-async-cats-effect](https://github.com/frees-io/freestyle/tree/master/modules/async/async-cats-effect/shared/src) Freestyle integration. This integration basically provides an `AsyncContext` for `cats.effect.IO`.

Add the following to the `commonSettings`:

```scala
"io.frees"  %% "frees-async-cats-effect" % freesV
```

We haven't finished with the [build.sbt](./build.sbt) file yet, since we need to add a new sbt module to place our RPC Server code:

```scala
// RPC Server.
lazy val server = project
  .in(file("server"))
  .settings(moduleName := "rpc-server")
  .settings(commonSettings)
  .aggregate(services)
  .dependsOn(services)
```

### RPC Server

All the code for this subsection will be placed at `server.scala` in [./server/src/main/scala/](./server/src/main/scala/).

* Concurrent Monad definition. In this case, we can add a type alias for simplicity. This might be placed inside a `package object`, for example:

```scala
package scalaexchange

import cats.effect.IO

package object serverapp {

  type ConcurrentMonad[A] = IO[A]

  val port: Int = 8080

}
```

* Implicit Runtime evidences. At the very least, we need to provide:
  * An `FSHandler` (Natural Transformation) instance of the service defined before.
  * Server configuration, where we will bind the `RFMAnalysis` service and configure the tcp port to `8080`.

  This is done at `implicits.scala` in [./server/src/main/scala/](./server/src/main/scala/)

```scala
package scalaexchange
package serverapp

import cats.{~>, Applicative}
import freestyle.rpc.server.handlers.GrpcServerHandler
import freestyle.rpc.server._
import freestyle.rpc.server.implicits._
import scalaexchange.services.protocol.RFMAnalysisService

import scalaexchange.services.runtime.RFMAnalysisServiceHandler

trait Implicits extends scalaexchange.CommonImplicits {

  implicit def rfmAnalisysServiceHandler[F[_]: Applicative]: RFMAnalysisServiceHandler[F] =
    new RFMAnalysisServiceHandler[F]

  val grpcConfigs: List[GrpcConfig] = List(
    AddService(
      RFMAnalysisService.bindService[ConcurrentMonad]
    )
  )

  implicit def grpcServerHandler: GrpcServer.Op ~> ConcurrentMonad =
    new GrpcServerHandler[ConcurrentMonad] andThen
      new GrpcKInterpreter[ConcurrentMonad](ServerW(port, grpcConfigs).server)

}
```

Notice `import freestyle.rpc.server.implicits._`, which is providing all we need to bootstrap the server, in terms of common async instances and so on.

* Finally, how would the **Server** look like?
  Have a look at `ServerApp.scala` in [./server/src/main/scala/](./server/src/main/scala/)


```scala
package scalaexchange
package serverapp

import freestyle._
import freestyle.rpc.server._
import freestyle.rpc.server.implicits._

object ServerApp extends scalaexchange.serverapp.Implicits {

  def main(args: Array[String]): Unit =
    server[GrpcServerApp.Op].interpret[ConcurrentMonad].unsafeRunSync()

}
```

To run the server, just type:

```bash
sbt server/run
```

### App Client Demo

Now, let's see how the auto-generated client can be used to invoke this service. We'll do this in our [app](./app/src/main/scala) sbt module.

First, let's define some instances needed to setup the client, like the port where the server is listening to the incoming connections.

Let's start with `implicits.scala` in [./app/src/main/scala/](./app/src/main/scala/)

```scala
package scalaexchange
package app

import cats.effect.IO
import freestyle.rpc.client._
import io.grpc.ManagedChannel

trait Implicits extends scalaexchange.CommonImplicits {

  val channelFor: ManagedChannelFor = ManagedChannelForAddress("localhost", 8080)

  val channelConfigList: List[ManagedChannelConfig] = List(UsePlaintext(true))

  val managedChannelInterpreter =
    new ManagedChannelInterpreter[IO](channelFor, channelConfigList)

  val channel: ManagedChannel = managedChannelInterpreter.build(channelFor, channelConfigList)

}
```

Then, let's use the auto-derived client based on the protocol definition of the service.

See `AppRFMClient.scala` in [./app/src/main/scala/](./app/src/main/scala/)

```scala
package scalaexchange
package app

import cats.effect.IO
import freestyle.rpc.protocol.Empty
import freestyle.asyncCatsEffect.implicits._
import freestyle.rpc.client.implicits._

import scalaexchange.services.protocol._

object AppRFMClient extends Implicits {

  def main(args: Array[String]): Unit = {

    implicit val rfmClient: RFMAnalysisService.Client[IO] =
      RFMAnalysisService.client[IO](channel)

    val result: SegmentList = rfmClient.segments(Empty).unsafeRunSync

    println(s"\n${result.list.mkString("\n")}\n")
  }

}
```

Keep in mind two important _imports_ that are making this program interpretation possible by the `ConcurrentMonad`, which is a `cats.effect.IO` type alias, as we mentioned earlier:

```
import freestyle.asyncCatsEffect.implicits._ // From frees-async-cats-effect
import freestyle.rpc.client.implicits._      // From frees-rpc
```

If you ran the server in the previous step, you can now test this client by typing the following command:

```bash
sbt "app/runMain scalaexchange.app.AppRFMClient"
```

## Server Streaming

What about streaming? As we already have a data-generator which returns a `Stream` of user events, let's expose it through our `RFMAnalysisService`, as a new RPC endpoint. In this case, we are talking about a kind of server streaming (in other words, `ResponseStreaming`).

* Before moving on, as we are using the data-generator, let's add it as a dependency in our `build.sbt` file:

```scala
// RPC definitions and implementations:
lazy val services = project
  .in(file("services"))
  .settings(moduleName := "rpc-services")
  .settings(commonSettings)
  .aggregate(`functional-microservices`, `data-generator`)
  .dependsOn(`functional-microservices`, `data-generator`)
```

* Now, let's add the new streaming service to our protocol definition. See `protocol.scala `in  [./services/src/main/scala/](./services/src/main/scala/)

```scala
@rpc(Avro)
@stream[ResponseStreaming.type]
def userEvents(empty: Empty.type): F[Observable[UserEvent]]
```

As you might notice, we are exposing a server streaming service that is returning a `monix.reactive.Observable` enclosing every user event that will be generated by our data-generator. Apart from this streaming returning type, we need to specify that we are creating a _ResponseStreaming_ service, by using the `@stream[ResponseStreaming.type]` annotation.

* Streaming Service implementation: we are basically done with this part since our data-generator is already using a `monix.reactive.Observable` of `UserEvent`.

Look at `RFMAnalysisServiceHandler.scala` in [./services/src/main/scala/runtime/](./services/src/main/scala/runtime/)

```scala
// Local instance of our data-generator streaming service:
private[this] val streamingService = new StreamingService

// ...

override def userEvents(empty: Empty.type): F[Observable[UserEvent]] =
  Applicative[F].pure(streamingService.userEventsStream)
```

* Finally, let's use this new service from our `app` client application. This would be an example.
  Have a look at `AppRFMClient.scala` in [./app/src/main/scala/](./app/src/main/scala/)

```scala
package scalaexchange
package app

import cats.effect.IO
import freestyle.rpc.protocol.Empty
import freestyle.asyncCatsEffect.implicits._
import freestyle.rpc.client.implicits._
import monix.reactive.Observable

import scala.concurrent.Await
import scala.concurrent.duration._
import scalaexchange.services.protocol._

object AppRFMClient extends Implicits {

  def main(args: Array[String]): Unit = {

    implicit val rfmClient: RFMAnalysisService.Client[IO] =
      RFMAnalysisService.client[IO](channel)

    val (segments: IO[SegmentList], stream: Observable[UserEvent]) =
      (rfmClient.segments(Empty), rfmClient.userEvents(Empty))

    println(s"Segments: \n${segments.unsafeRunSync().list.mkString("\n")}\n")
    Await.ready(
      stream
        .map { u =>
          println(u)
          u
        }
        .completedL
        .runAsync,
      Duration.Inf)
  }

}
```

This was super-easy, wasn't it? Now, we are going to test it 'end2end':

Run Server:

```bash
sbt server/run
```

Run Client Application:

```bash
sbt "app/runMain scalaexchange.app.AppRFMClient"
```

The expected result is that both server and client console outputs will show these generated user events.

## Client Streaming

So far, we've seen how to implement RPC services in two different fashions: unary and server streaming. Now, we are going to see how to implement a new type of service where the client is the one sending a stream of data. Concretely, and for demo purposes in this workshop, think about a server endpoint able to store `Orders` somewhere that any client is sending them in a streaming way.

* As usual, let's add some new settings to our `build.sbt` file. For the sake of simplicity, we are adding a couple of new dependencies to our `commonSettings` val:

```scala
// ... libraryDependencies ++= Seq(
"joda-time" % "joda-time" % "2.9.9",
"com.github.tototoshi" %% "scala-csv" % "1.3.5"
//...
```

* Let's define a new model for orders, which we could place at [./src/main/scala/models.scala](./src/main/scala/models.scala):

```scala
case class CustomerData(date: String, orderId: String, total: Int)

case class Order(customerId: Int, data: CustomerData)
```

* Moving on, we are now prepared to add this new client streaming service to the protocol definition at [./services/src/main/scala/protocol.scala](./services/src/main/scala/protocol.scala)

```scala
// ... Add a sample response message as ack of this new service
final case class Ack(result: String)

// ...

@rpc(Avro)
@stream[RequestStreaming.type]
def orderStream(orders: Observable[Order]): F[Ack]
```

As you can see, now the streaming type (`Observable`) is in the request. Also, notice the annotation in this case: `@stream[RequestStreaming.type]`.

* Next, let's add the implementation for this new service where, in this case, we will consume the stream of `Order`s writing it to an `CSV` (Comma-separated values) file. The following code shows how [./services/src/main/scala/runtime/RFMAnalysisServiceHandler.scala](./services/src/main/scala/runtime/RFMAnalysisServiceHandler.scala) would look like after implementing the `orderStream` service.

```scala
package scalaexchange
package services
package runtime

import java.io.File

import cats.{~>, Applicative}
import com.github.tototoshi.csv._
import freestyle.rpc.protocol._
import monix.eval.Task
import monix.execution.Scheduler
import monix.reactive.Observable

import scalaexchange.datagenerator.StreamingService
import scalaexchange.services.protocol._

class RFMAnalysisServiceHandler[F[_]: Applicative](implicit S: Scheduler, T2F: Task ~> F)
    extends RFMAnalysisService[F] {

  private[this] val segmentList: List[Segment] = List(
    Segment("Champions", 4, 5, 4, 5, 4, 5),
    Segment("Loyal Customers", 2, 5, 3, 5, 3, 5),
    Segment("Potential Loyalist", 3, 5, 1, 3, 1, 3),
    Segment("New Customers", 4, 5, 0, 1, 0, 1),
    Segment("Customers Needing Attention", 3, 4, 0, 1, 0, 1),
    Segment("About To Sleep", 2, 3, 0, 2, 0, 2),
    Segment("Can't Lose Them", 0, 1, 4, 5, 4, 5),
    Segment("At Risk", 0, 2, 2, 5, 2, 5),
    Segment("Hibernating", 1, 2, 1, 2, 1, 2),
    Segment("Lost", 0, 2, 0, 2, 0, 2)
  )

  private[this] val streamingService = new StreamingService

  private[this] val outPath: String = "orders.csv"

  override def segments(empty: Empty.type): F[protocol.SegmentList] =
    Applicative[F].pure(SegmentList(segmentList))

  override def userEvents(empty: Empty.type): F[Observable[UserEvent]] =
    Applicative[F].pure(streamingService.userEventsStream)

  override def orderStream(orders: Observable[Order]): F[Ack] = T2F {
    val f: File           = new File(outPath)
    val writer: CSVWriter = CSVWriter.open(f)

    orders
      .foreachL { order =>
        writer.writeRow(
          List(order.data.date, order.data.orderId, order.customerId, order.data.total)
        )
      }
      .map { _ =>
        writer.close()
        Ack(" 👍 ")
      }
  }
}
```

Now, a few new things are needed for the `orderStream` service implementation:

- A CSV Writer for Scala, in this case we are using: https://github.com/tototoshi/scala-csv. However, this order stream could be stored somewhere else, like a database or sent to a kafka topic, among other choices.
- In the class constructor, we are requiring a new implicit dependency (`T2F: Task ~> F`), which is a natural transformation to go from `monix.eval.Task` to our `F`. This is needed because of the way we are consuming the `monix.reactive.Observable` through the `foreachL` (it creates a new `monix.eval.Task Task` that will consume the observable, executing the given callback for each element).

* Actually, we need to specify the same implicit evidence requirement when instantiating the `RFMAnalysisServiceHandler`, at class [scalaexchange.serverapp.Implicits](./server/src/main/scala/implicits.scala):

```scala
implicit def rfmAnalisysServiceHandler[F[_]: Applicative](
    implicit T2F: monix.eval.Task ~> F): RFMAnalysisServiceHandler[F] =
  new RFMAnalysisServiceHandler[F]
```

Fortunately, we don't need to implement anything else because `frees-rpc` is providing for free this natural transformation when importing `freestyle.rpc.server.implicits._`.

* To finish this section, let's update our client application with the new server version, where we can invoke the `orderStream` service that sends a stream of orders. In order to create the stream, we are using scalacheck together with [scalacheck-toolbox](https://github.com/47deg/scalacheck-toolbox) to generate random dates for a specific period of time.

See [./app/src/main/scala/AppRFMClient.scala](./app/src/main/scala/AppRFMClient.scala)


```scala
package scalaexchange
package app

import cats.effect.IO
import freestyle.rpc.protocol.Empty
import freestyle.asyncCatsEffect.implicits._
import freestyle.rpc.client.implicits._
import monix.reactive.Observable

import scala.concurrent.Await
import scala.concurrent.duration._
import scalaexchange.services.protocol._

object AppRFMClient extends Implicits {

  def main(args: Array[String]): Unit = {

    implicit val rfmClient: RFMAnalysisService.Client[IO] =
      RFMAnalysisService.client[IO](channel)

    val (segments: IO[SegmentList], stream: Observable[UserEvent], ack: IO[Ack]) =
      (
        rfmClient.segments(Empty),
        rfmClient.userEvents(Empty),
        rfmClient.orderStream(ordersStreamObs)
      )

    println(s"Segments: \n${segments.unsafeRunSync().list.mkString("\n")}\n")
    println(s"Client Streaming: \n${ack.unsafeRunSync()}\n")
    Await.ready(
      stream
        .map { u =>
          println(u)
          u
        }
        .completedL
        .runAsync,
      Duration.Inf)
  }

  private[this] def ordersStreamObs: Observable[Order] = {
    val orderList: List[Order] = (1 to 1000).map { customerId =>
      import com.fortysevendeg.scalacheck.datetime.GenDateTime
      import org.joda.time.{DateTime, Period}
      import org.scalacheck._
      import com.fortysevendeg.scalacheck.datetime.instances.joda.jodaForPeriod

      (for {
        date    <- GenDateTime.genDateTimeWithinRange(DateTime.parse("2017-12-01"), Period.days(22))
        orderId <- Gen.uuid
        total   <- Gen.choose[Int](5, 200)
      } yield
        Order(
          customerId,
          CustomerData(date.toString, orderId.toString, total)
        )).sample.get
    }.toList

    Observable.fromIterable(orderList)
  }

}
```

And that's it for now. To test this out, run the server:

```bash
sbt server/run
```

And run the new client application version:

```bash
sbt "app/runMain scalaexchange.app.AppRFMClient"
```

The expected result for the steps completed in this section would be that you will find an `orders.csv` file within your project, which will contain all the orders sent via streaming from the client. Pat your back for reaching this point.

## What's Next

* Bi-directional streaming.
* Complete the RFM Analysis example through new RPC microservices.
