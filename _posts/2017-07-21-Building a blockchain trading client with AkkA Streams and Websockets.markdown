---
layout: default
title:  "Building a crypto currency trading client with Akka Streams and WebSockets"
date:   2017-07-21 
categories: tech
---
# Crypto asset platform client for Gemini with Akka

Being interested in distributed ledger technology and actively trading crypto currency I'm always trying to make my life easier with tooling. There are several crypto trading platforms and most of them are providing an API.

Today I want to talk about how we can use the WebSocket API provided by [Gemini](http://gemini.com) to get market data in [near real time](https://en.wikipedia.org/wiki/Real-time_computing) that can be utilized for a better overview over the market and potentially much more.

Gemini offer two different types of [WebSocket API](https://docs.gemini.com/websocket-api/):

- private Order Events API
- public Market Data API

this post, we cover the latter one.

## Problem

We need a program that allows us to

- Subscribe to the event stream
- Lift the events inside our intern data model
- Be able to extract information by applying further transformations, filters, aggregations etc... on the stream
- Can be composed with data streams of other exchanges for comparisons or decisions based on different sources

Sounds good so far, although the requirements bring a whole set of engineering problems if we want to have a stable piece of software. Special attention should be granted to IO and network aspects as well as to concurrency and flow control.

In detail we have to consider:

- Failures on the producer side of the event stream e.g. event stream slows down or drops
- Failures on the network layer
- Concurrently process data with respect to available heap memory
- IO does not block CPU heavy computations and vice versa

Sounds a lot, and it would be a lot, if most of it not came for free :) with [akka-streams](http://doc.akka.io/docs/akka/snapshot/scala/stream/index.html). Akka-strams providing out-of-the box support for _concurrent stream processing_, flow control mechanisms like _buffering_ or _backpressure_, a DSL to _express streams as graphs_ and many more cool things you can check on their [ homepage](http://doc.akka.io/docs/akka/snapshot/scala/stream/stream-quickstart.html).

The libraries we will use:
- [akka-http 10.0.7](http://doc.akka.io/docs/akka-http/current/scala/http/)
- [akka-stream 2.5.3](http://doc.akka.io/docs/akka/snapshot/scala/stream/index.html)
- [circe 0.8.0](https://circe.github.io/circe/)

Let's get this party started!

## Subscribing the event stream

Before doing something with data - we have to get data!

Akka-streams is giving us a build-in support for WebSockets. Let's use it to construct a _request flow_.

{% highlight scala %}
// Source address
val uri = "wss://api.gemini.com/v1/marketdata/"
val pair = "ethusd"
// Constructing the flow for ethusd
val requestFlow : Flow[Message, Message, Future[WebSocketUpgradeResponse]] =
  Http().webSocketClientFlow(WebSocketRequest(s"$uri/$pair"))
{% endhighlight %}

The _requestFlow_ produce an output of type `Message` that is representing a message in context of WebSocket.

An akka stream usually contains a `Source ~> Flow ~> Sink`

![SourceSinkFlow](https://opencredo.com/wp-content/uploads/2015/10/source-flow-sink.png)
{:.image-caption}
*Image is taken from opencredo.com*
{:.image-caption}

in our case we would like to have our event stream represented by a Source for example as a `Source[Event]` based on that we can compose flows that could _extract/filter/aggregate_ event information and send it to a custom Sink that finish out stream by dumping the data _inside a database_ or to _another service_.

Another important case is that the Gemini API do support a market data stream for 3 different pairs of currency.


| From        | To           | Name  |
| ------------- |:-------------:| -----:|
| Ethereum      | USD | ethusd |
| Bitcoin      | USD      |   btcusd |
| Ethereum | Bitcoin      |    ethbtc |


We will combine them inside one Flow to maximum avoid boilerplate code.

{% highlight scala %}
// Define our Source
val ourSource = Source.fromGraph(GraphDSL.create() { implicit b =>
    import GraphDSL.Implicits._

    // Create requests for all currency pairs available at Gemini
    val requests = CurrencyPairs.values.map { currencyPair =>
      Http().webSocketClientFlow(WebSocketRequest(s"$uri/$currencyPair")
    }

    // Source with empty Promise[Message]
    val source = Source.maybe[Message]

    // Create a merge shape that combine the request flows
    val merge = b.add(Merge[Message](requests.size))

    // Merge the flows  
    requests.zipWithIndex.foreach { case (flow, i) =>
      source ~> flow ~> merge.in(i)
    }
    // Return the merge inside a SourceShape
    SourceShape(merge.out.outlet)
  })
{% endhighlight %}


Ok, let's look into the above statement in detail:
1. We are creating a `Source` from a `Graph` described with `GraphDSL`, a Domain Specific Language in akka that allow you to write your processes as a graph.
2. Creating a `Flow[Message, Message, Future[WebSocketUpgradeResponse]]` for each currency pair.
3. Declare a `Source.maybe[Message]` as kind of a trigger for our WebSocket flow. It
sends a `Promise[Message]` downstream. As long we not complete the `Promise[Message]` the stream will not complete and we keep listening the incoming. This technique provide us with an [infinite incoming stream](http://doc.akka.io/docs/akka-http/current/scala/http/client-side/websocket-support.html).
4. We combine the currency pair flows from step 2 in a so called _junction_. The junction of type `Merge[In]` combine _N inputs to 1 output_.
5. Define the output of merge as the output of the `Source`.

## Get the data model straight

To work on data - we have to interpret data!

Now as we have the event stream, we should lift the incoming `Message` inside an ADT. Gemini sends us JSON, as next,
we define our ADT by simply writing types that are [mirroring the responses](https://docs.gemini.com/websocket-api/#market-data). Let's have a brief look how it might look like:

{% highlight scala %}
// represent one Message
case class GeminiEvent(eventId: Long, events: Seq[GeminiEventDelta],
                       currencyPair: Option[CurrencyPair] = None
                      ) extends MarketEvent

// Marker for different types of delta events.
sealed trait GeminiEventDelta {
  // Geminis event type marker
  val eventType: GeminiEventType
}
// An example how a delta event looks like
case class ChangeEvent(
                        delta: BigDecimal,
                        price: BigDecimal,
                        reason: GeminiEventReason,
                        remaining: BigDecimal,
                        side: GeminiSide
                      ) extends GeminiEventDelta {
  override val eventType: GeminiEventType = GeminiEventTypes.change

  // All type definitions can be found in git.
  //https://github.com/allquantor/scemini/blob/master/src/main/scala/io.allquantor.s//cemini/adt/gemini/GeminiEvents.scala
}
{% endhighlight %}

Events also contain some constant expressions that in our implementation are expressed with case objects. An implementation with case objects instead of `scala.Enumeration` is generally more flexible and typesafe, since enums are [erasing type information after compiling](http://underscore.io/blog/posts/2014/09/03/enumerations.html).

{% highlight scala %}
object GeminiEventTypes extends Stringable {

  sealed trait GeminiEventType

  case object trade extends GeminiEventType
  case object change extends GeminiEventType
  case object auction_indicative extends GeminiEventType
  case object auction_open extends GeminiEventType
  case object auction_result extends GeminiEventType

  case object geminiTypeDefault extends GeminiEventType

  val values = Set(trade, change, auction_indicative, auction_open, auction_result)

  override type T = GeminiEventType
  override val defaultValue: GeminiEventType = geminiTypeDefault

  // All type definitions can be found in git.
  //https://github.com/allquantor/scemini/blob/master/src/main/scala/io.allquantor.sce//mini/adt/gemini/GeminiConstants.scala
}

{% endhighlight %}

## Lift the Data

What have we done so far? We subscribe the stream data and know how the model looks like. What is missing is the part `Message => GeminiEvent` - making a type out of the incoming message.

There are numerous libraries in Scala that work with JSON. Although akka have a build in integration with  [spray-json](http://doc.akka.io/docs/akka-http/10.0.7/scala/http/common/json-support.html) I will use [circe](https://circe.github.io/circe/) which is a pretty neat pure functional library for working with JSON.

Circe handle the decoding of JSON the same as almost all modern JSON libraries - by using [Ad hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism). We have to provide an implicit typeclass inside the context of JSON deserialization.

This is how it looks like:

{% highlight scala %}
trait GeminiMarketReads {

  // Parsing recursion entry point.
  implicit val eventRead: Decoder[GeminiEvent] = (c: HCursor) => for {
    id <- c.downField("eventId").as[Long]
    events <- c.downField("events").as[Seq[GeminiEventDelta]]
  } yield GeminiEvent(id, events)

  // Identify the event type and run the appropriate reader.
  implicit val eventDeltaRead: Decoder[GeminiEventDelta] = (c: HCursor) => {

    implicit val cursor = c
    val eventType = c.downField("type").to(GeminiEventTypes.fromString)

    eventType.flatMap {
      case GeminiEventTypes.change => changeEventRead
      case GeminiEventTypes.trade => tradeEventRead
      case GeminiEventTypes.auction_open => auctionOpenRead
      case GeminiEventTypes.auction_indicative => auctionIndicativeRead
      case GeminiEventTypes.auction_result => auctionResultRead
      case _ => Decoder.failedWithMessage("Event type is unknown, JSON could not be decoded")(c)
    }
  }
  // All decoders can be found in git
  //https://github.com/allquantor/scemini/blob/master/src/main/scala/io.allquantor.scemini/materialization/GeminiMarketReads.scala
}
{% endhighlight %}

We get the `eventID` and the `EventType` and forward the `c:HCursor` to the appropriate decoder instance. Decoder yield an instance of  `Either[circe.Error,T]` where `T` is the expected result type. The decoding is side effect free, any parsing/decoding error is gonna be encoded as `circe.Error` and will contain a sequence of actions done on your JSON before any fail and you get a good understanding where and why it has failed.


## Integrating parsing

Lets define a `Flow` of type `Flow[(Message,CurrencyPair), Future[Either[circle.Error, GeminiEvent]]]` where each element is a tuple representing a `WebSocket` message and the according `CurrencyPair` and transform it to the event type of our model.

{% highlight scala %}

private val lifting = {

   // Define a separate thread pool for CPU intensive operations.
   val CPUec = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(2))

    // Helper to avoid code duplication
    implicit class Parser(s: String) {
      def transform(implicit c: CurrencyPair) = parse(s)
        .flatMap(_.as[GeminiEvent])
        .map(_.copy(currencyPair = Some(c)))
    }

    Flow[(Message, CurrencyPair)].collect {
      case (elem@(m: Message, c: CurrencyPair)) =>
        implicit val currencyPair: CurrencyPair = c
        (m: @unchecked) match {
          // Using a separate fixed thread pool for potentially CPU intensive computations
          case TextMessage.Strict(msg) => Future(msg.transform)(CPUec)
          case TextMessage.Streamed(stream) => stream.limit(100)
            .completionTimeout(5000.millis)
            .runFold("")(_ + _).map(_.transform)(CPUec)
        }
    }.mapAsync(4)(identity)
  }
{% endhighlight %}

A WebSocket Message have two subtypes

1. `Strict` - the message data is already available as a whole.
2. `Streamed` - the data is a `Source` streaming the data comes in.

Gemini use both of them. `Streamed` as the potentially large initial JSON that mirror the entire state of the order book and `Strict` for smaller update events.

As soon as we have the message we parse it into `circe` intern AST structure and then decode it in to our model. The parsing of JSON can be CPU intensive, to ensure CPU and IO tasks does not stand in each others way we are using a separate fixed thread pool exclusively for decoding operations. Since our flow return a Future we use [mapAsync](http://doc.akka.io/docs/akka/current/scala/stream/stages-overview.html#mapasync) to process the tasks concurrency and pass them downstream.

## Put everything together

Lets consolidate our implementations inside a class.

{% highlight scala %}
class GeminiMarkets(currencyPairs: Seq[CurrencyPair],
                    uri: String)(implicit system: ActorSystem)
  extends ExchangePlatformClient with GeminiMarketReads {

  private val requests = currencyPairs.map(pair =>
    (Http().webSocketClientFlow(WebSocketRequest(s"$uri/$pair")), pair))

  private val lifting = {

    implicit class Parser(s: String) {
      def transform(implicit c: CurrencyPair) = parse(s)
        .flatMap(_.as[GeminiEvent])
        .map(_.copy(currencyPair = Some(c)))
    }
    Flow[(Message, CurrencyPair)].collect {
      case (elem@(m: Message, c: CurrencyPair)) =>
        implicit val currencyPair: CurrencyPair = c
        (m: @unchecked) match {
          case TextMessage.Strict(msg) => Future(msg.transform)(CPUec)
          case TextMessage.Streamed(stream) => stream.limit(100)
            .completionTimeout(5000.millis)
            .runFold("")(_ + _).map(_.transform)(CPUec)
        }
    }.mapAsync(4)(identity)
  }

  lazy val source = Source.fromGraph(GraphDSL.create() { implicit b =>
    import GraphDSL.Implicits._

    val source = Source.maybe[Message]

    val merge = b.add(Merge[(Message, CurrencyPair)](requests.size))

    requests.zipWithIndex.foreach { case ((flow, c), i) =>
      source ~> flow.map((_, c)) ~> merge.in(i)
    }
    SourceShape(merge.out.via(lifting).outlet)
  })

  implicit val system = ActorSystem()
  implicit val mat = ActorMaterializer.create(system)
  implicit val ec = system.dispatcher
  val logger = Logging.getLogger(system, this)

  lazy val supervisionStrategy: Supervision.Decider = {
    case ex: Exception =>
      logger.error(s"Error in graph processing with ${ex.getStackTrace.mkString(",")}")
      Supervision.Resume
  }

  val currencyPairs = Seq(CurrencyPairs.ethusd, CurrencyParis.btcusd)
  // Use the abstract class to hide some instance creation logic
  val client = ExchangePlatformClient.asGeminiCleint(currencyPairs)
    .source
    // Define recover strategy when an exception is thrown inside of the source
    .withAttributes(ActorAttributes.supervisionStrategy(supervisionStrategy))
  client.to(Sink.foreach(println)).run()
}
{% endhighlight %}

Beside of consolidation, this code emphasize an approach for the failure handling problem defined above.

We implemented a `supervisionStrategy: Supervision.Decider` that decide how do deal with exceptions thrown inside the stream.

In our case the decider will log the stackstrace and resume with the stream processing - emit the next element. Akka offers three ways to react on exceptions: `Stop, Resume, Restart`, please check the [documentation](http://doc.akka.io/docs/akka/current/scala/stream/stream-error.html)to find a strategy which fits your needs.

## Summary

As result we have implemented a Source of market events that now can be used for  advanced analytical tasks. It has been shown that the usual set of problems coming with streaming events from an external source can be successfully mitigated with the toolset akka-streams provide.

Now, we could use akkas flow processing functionality to extract data of our interest.

A simple example if we only interested on `TradeEvents`

{% highlight scala %}
val dumpIntoDataBaseSink = Sink.foreach(...)

source.filter(_.isRight)
.map(_.right.get)
.map(_.events.filter(_.eventType == GeminiEventTypes.trade))
.to(dumpIntoDataBaseSink)
.run()
{% endhighlight %}

We can combine it with streams from other exchanges, or apply advanced techniques like windowing to get time based information.

The code is [available on Github](https://github.com/allquantor/scemini). Thanks for reading and happy hAkking!
