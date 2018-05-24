# KeenClient-Scala

[![Build Status]](https://travis-ci.org/keenlabs/KeenClient-Scala)
[![Coverage Status]](https://coveralls.io/github/keenlabs/KeenClient-Scala?branch=master)
[![Maven Central Badge]](https://maven-badges.herokuapp.com/maven-central/io.keen/keenclient-scala_2.11)
[![Scaladoc Badge]](https://keenlabs.github.io/KeenClient-Scala/)

---

The official asynchronous Scala client for the [Keen IO] API.

**Note**: This library is in early development and does not implement all of the
features of the Keen API. It is pre-1.0 in the [Semantic Versioning] sense:
public interfaces may change without backwards compatibility. We will try to
minimize breaking changes, but please consult [the changelog] when updating to a
new release version.

Additional API features will be added over time. Contributions are welcome!

---

* [Get It](#get-it)
* [Configuration](#configuration)
  * [Settings](#settings)
* [Use It](#use-it---a-quick-taste)
  * [Batching Write Requests](#batching-write-requests)
* [Dependencies](#dependencies)
  * [Using the Dispatch adapter](#using-the-dispatch-adapter)
  * [JSON](#json)
* [Hack On It](#hack-on-it)


## Get It

Artifacts for keen-client-scala are [hosted on Maven Central](http://search.maven.org/#search%7Cga%7C1%7Ckeenclient-scala).
You can use them in your project with SBT thusly:

```scala
libraryDependencies += "io.keen" %% "keenclient-scala" % "0.7.0"
```

Note that we publish artifacts for Scala 2.10 and 2.11, so you can either use `%%` to automatically pick the correct
version or specify them explicitly with something like:

```scala
libraryDependencies += "io.keen" % "keenclient-scala_2.10" % "0.7.0"
```

## Configuration

The client has a notion of access levels that reflect [the Keen IO API key
security model][security]. These are represented by Scala traits called
`Reader`, `Writer`, and `Master`. According to the level of access that your
application requires, you must mix the appropriate trait(s) into your client
instance when creating it, and configure your corresponding API keys. This is
demonstrated in the examples.

Configuration is supported by the [Typesafe config] library, offering all the
flexibility you could wish to provide settings through a file, environment
variables, or programmatically. We recommend environment variables for your API
keys at very least, to avoid storing credentials in source control. To that end,
the following will be honored by default if set:

* `KEEN_PROJECT_ID`
* `KEEN_READ_KEY`
* `KEEN_WRITE_KEY`
* `KEEN_MASTER_KEY`

To configure with a file, it must be on the classpath, customarily called
`application.conf` though you may use others--see the Typesafe config
documentation for all the options and details of the file format. [Our
`reference.conf`] reflects all of the settings you can configure and their
default values.

For advanced needs, you may provide your own custom [`Config`] object by simply
passing it to the client constructor:

```scala
import io.keen.client.scala.Client

val keen = new Client(config = myCustomConfigObject)
```

When using environment variables, you might like [sbt-dotenv] in your
development setup (install it as a [global plugin], and `chmod 600` your `.env`
files that contain credentials!). In production, a [good service manager][runit]
can set env vars for app processes with ease. On Heroku you'll be right at home.

### Settings

* `keen.project-id`: Your project ID.
* `keen.optional.read-key`: Your project read key.
* `keen.optional.write-key`: Your project write key.
* `keen.optional.master-key`: Your project master key.
* `keen.queue.batch.size`: Number of events to include in each batch sent by `sendQueuedEvents()`. Default is `500`.
* `keen.queue.batch.timeout`: Duration that each batch sent by `sendQueuedEvents()` should wait before the request times out. Default is 5 seconds.
* `keen.queue.max-events-per-collection`: Maximum number of events to store for each collection. Old events are purged from the queue to make room for new events when the size of the queue exceeds this number. Default is `10000`.
* `keen.queue.send-interval.events`: Automatically send all queued events every time the queue reaches this number. Minimum is `100`, maximum is `10000`, and default is `0`.
* `keen.queue.send-interval.duration`: Automatically send all queued events at a specified interval. Minimum is 60 seconds, maximum is 3600, and default is 0.
* `keen.queue.shutdown-delay`: Duration before client stops attempting to send events scheduled to be sent at a specific interval. Default is 30 seconds.

## Use It - A Quick Taste

```scala
import io.keen.client.scala.{ Client, Writer }

// Assumes you've configured a write key as explained in Configuration above
val keen = new Client with Writer

// Publish an event!
keen.addEvent(
  collection = "collectionNameHere",
  event = """{"foo": "bar"}"""
)

// Publish lots of events!
keen.addEvents(someEvents)

// Responses are Futures - handle errors!
val resp = keen.addEvent(
  collection = "collectionNameHere",
  event = """{"foo": "bar"}"""
)

resp onComplete {
  case Success(r) => println(resp.statusCode)
  case Failure(t) => println(t.getMessage) // A Throwable
}

// Or using map
resp map { println("I succeeded!") } getOrElse { println("I failed :(") }
```

### Batching Write Requests

The standard client is asynchronous, but it does make a discrete API call for each individual (`addEvent`) or bulk (`addEvents`) event write. Depending on your usage patterns, batching events to make fewer HTTP requests to Keen IO may be a significant optimization.

Though you can certainly implement your own queueing and batching via `addEvents`, the library includes automated batching support sufficient for many use cases. This is available in the `BatchWriterClient`, shown below.

```scala
import io.keen.client.scala.BatchWriterClient

val keen = new BatchWriterClient

// Queue an event
keen.queueEvent("collectionNameHere", """{"foo": "bar"}""")
```

Queuing is handled via an in-memory queue that lives as long as the `Client` does. The behavior of the queue and automated sending of events is configurable in `conf/application.conf` as outlined below.

**Sending queued events manually**

```scala
keen.sendQueuedEvents()
```

**Sending queued events every time the queue reaches 100 events**

Set `keen.queue.send-interval.events` equal to `100` in `conf/application.conf`.

**Sending queued events every 5 minutes**

Set `keen.queue.send-interval.duration` equal to `5 minutes` in `conf/application.conf`.

Note that `send-interval.events` takes precedence when both `send-interval.events` and `send-interval.duration` contain values greater than zero.

**Using batch sizes**

Setting a specific batch size will help optimize your experience when sending events. It's recommended that you set `keen.queue.batch.size` to something that makes sense for your application (default is `500`). Note that a batch size of `5000` is the upper bound of what you should shoot for. Anything higher and your request has a good chance of being rejected due to payload size limitations.

**Failed events**

Events that fail to be sent for whatever reason (payload rejection, network issue, etc.) are **not** removed from the queue. They will remain in the queue until they are either manually removed or the client shuts down.

**Shutdown**

`BatchWriterClient` will attempt one last time to send all queued events when `shutdown()` is called just in case you forget to send the events yourself. Make sure you call `shutdown()` before you exit otherwise all events that remain in the queue upon termination will be lost.

#### Caveats and Tradeoffs

Bear in mind that `BatchWriterClient` uses an in-memory event queue implementation by default and has an API that favors fire-and-forget usage; it is difficult to handle errors in a granular fashion and ensure that writes are never lost if queue bounds are exceeded, etc. If you're recording something like internal metrics, the possibility of occasional missing data points may be an acceptable tradeoff for the optimization of batching.

If it is business-critical that some of your writes are never lost, you may wish to use the discrete API so that you can handle the `Future` result of each call with exacting care. Or if you require batching, you can implement a custom persistent `EventStore` queue for control over retries, and/or your own batching `Client` subclass tailored to your needs.

`BatchWriterClient` is a `Client with Writer`, so you can call the discrete `addEvent`/`addEvents` methods on an instance and as usual these send immediately without queueing. Thus you can selectively use batching for some types of writes and make discrete calls for more critical ones.

## Dependencies

The client's default HTTP adapter is built on the [spray HTTP toolkit][spray],
which is [Akka]-based and asynchronous. A [Dispatch]-based adapter is also
available. At this time spray (and thus Akka) is a hard dependency, but if there
is demand we may consider designating it as "provided" so that you may opt for
the Dispatch adapter (or a custom one for your preferred HTTP client) and avoid
pulling in Akka dependencies if you wish, or to avoid version conflicts. Please
share your feedback if you find the spray deps burdensome.

With either adapter, API calls will return a uniform `Future[Response]` type,
where `Response` is an `io.keen.client.scala.Response`. Instances have
`statusCode` and `body` attributes that you may inspect to act on errors. An
example of choosing the Dispatch adapter is shown below.

The client also depends on [grizzled-slf4j] for logging.

It is cross-compiled for 2.11 and 2.12 Scala versions. If you are interested in
support for other versions or discover any binary compatibility problems, please
share your feedback.

### Using the Dispatch adapter

To use the Dispatch HTTP adapter instead of the Spray default, specify the
following override when instantiating a `Client` instance:

```scala
import io.keen.client.scala.{ Client, HttpAdapterDispatch }

val keen = new Client {
  override val httpAdapter = new HttpAdapterDispatch
}
```

`BatchWriterClient` works the same way.

### JSON

Presently this library does **not** do any JSON parsing. It works with strings only. It is
assumed that you will parse the JSON returned and pass stringified JSON into any methods that
require it. Feedback is welcome!

We understand that Scala users value the language's strong type system. Again,
we wish to avoid unwanted dependencies given that there are so many JSON parsing
libraries out there. We'd eventually like to offer rich types through JSON
adapters with optional deps.

## Hack On It

Unit tests can be run with the standard SBT commands `test`, `testQuick`, etc.

The test suite includes integration tests which require keys and access to Keen
IO's API. If you have set keys through environment variables or configuration as
described above, you may run these with:

    $ sbt it:test

**Only use a dedicated dummy account for this purpose, data could be destroyed
that you didn't expect!**


[Build Status]: https://travis-ci.org/keenlabs/KeenClient-Scala.svg?branch=master
[Coverage Status]: https://coveralls.io/repos/github/keenlabs/KeenClient-Scala/badge.svg?branch=master
[Maven Central Badge]: https://maven-badges.herokuapp.com/maven-central/io.keen/keenclient-scala_2.11/badge.svg
[Scaladoc Badge]: https://img.shields.io/badge/scaladoc-latest-lightgrey.svg
[Keen IO]: http://keen.io/
[Semantic Versioning]: http://semver.org/
[the changelog]: https://github.com/keenlabs/KeenClient-Scala/blob/master/CHANGELOG.md
[spray]: http://spray.io
[Akka]: http://akka.io
[Dispatch]: http://dispatch.databinder.net/
[grizzled-slf4j]: http://software.clapper.org/grizzled-slf4j/
[security]: https://keen.io/docs/security/
[Typesafe config]: https://github.com/typesafehub/config
[Our `reference.conf`]: https://github.com/keenlabs/KeenClient-Scala/tree/master/src/main/resources/reference.conf
[`Config`]: http://typesafehub.github.io/config/latest/api/com/typesafe/config/Config.html
[sbt-dotenv]: https://github.com/mefellows/sbt-dotenv
[global plugin]: http://www.scala-sbt.org/0.13/docs/Global-Settings.html
[runit]: http://smarden.org/runit/
