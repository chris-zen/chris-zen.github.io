---
layout: post
title:  "ElasticSearch with Scala and Akka"
date:   2015-05-10 21:30:00
categories: software
tags: elasticsearch scala akka
---
# Introduction
I am working in a project that involves Elasticsearch, Scala and Akka,
and surprisingly for me, I have run into some problems not easily solvable
by a simple Google search, so I decided to put some notes here.

The project is about collecting logs and doing some analytics with the data.
There are several background services (implemented as Akka actors) which are
processing data from/to an Elasticsearch database.

# Choosing the client library

There are several [client libraries for Scala](http://www.elastic.co/guide/en/elasticsearch/client/community/current/clients.html#community-scala) which do a great job in integrating
Elasticsearch API into the Scala language, but after taking a look I decided
not to use any of them, and simply use the [native Java client](http://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html).
Before you blame on me, please hear my justification. I was mainly interested in
learning the basics of the Elasticsearch API and its core design principals so I
decided to use the library written in Java. The Scala versions are really cool,
but they add another layer on top of it that makes more
difficult to understand what is going on and solve problems.

# Connecting

There are several options to have an Elasticsearch client:

* Creating a local embedded node
* By creating a node that is part of a cluster and contributes with data
* By creating a node that doesn't contributes with data
* By creating a transport that just communicates requests and responses to
other nodes in the cluster.

I just needed to connect to an existing cluster without contributing in data,
and decided to use the third option. At the beginning I was confused where to
create the connection,  whether from the actors or somewhere else, how many
connections could I have,  was it thread safe ?  I had previous experience with
the Python client, but this was different, the  Python library is more like
using a transport where you can create several  client connections (and the
library maintains a pool of connections internally).  In this case, as the node
creation is a very expensive operation and the native Java client is thread
safe, it is better to create one instance during the application start and pass it
to the actors that require  access to the database (still not 100% sure if this
is the best pattern, but currently it works for me).

# Futures vs ActionListener

The native Java client does a good job with not blocking during the execution of
operations but the first thing I was missing was its integration with the Scala
futures. After browsing a bit I found [a good solution](http://loads.pickle.me.uk/2013/11/16/scala-futures-for-elasticsearch-java-api.html) to wrap the execution into a Scala future. After minor modifications, this is what I use:

{% highlight scala %}
import org.elasticsearch.action.{ActionRequestBuilder, ActionListener, ActionResponse}
import scala.concurrent.{Future, Promise}

object RequestExecutor {
    def apply[T <: ActionResponse](): RequestExecutor[T] = {
        new RequestExecutor[T]
    }
}

class RequestExecutor[T <: ActionResponse] extends ActionListener[T] {
    private val promise = Promise[T]()

    def onResponse(response: T) {
        promise.success(response)
    }

    def onFailure(e: Throwable) {
        promise.failure(e)
    }

    def execute[RB <: ActionRequestBuilder[_, T, _, _]](request: RB): Future[T] = {
        request.execute(this)
        promise.future
    }
}
{% endhighlight %}

And this is an example of use:

{% highlight scala %}
val req = esclient.prepareSearch()

val respFuture = RequestExecutor[SearchResponse].execute(req)

val hits = respFuture.map { response =>
    response.getHits.getHits
}
{% endhighlight %}

# Real life example with actors

Let's see a real life example on how to do a search without blocking an actor.
The following code have an actor that does an aggregation by timestamp when
receives a StartQuery message and returns a QueryResult with the list of
intervals found in the elasticsearch query result.

{% highlight scala %}
package example

import scala.concurrent.Future
import akka.actor.{Actor, ActorLogging, Props}
import akka.event.LoggingReceive
import akka.pattern.pipe
import org.elasticsearch.action.search.{SearchResponse, SearchType}
import org.elasticsearch.client.Client
import org.elasticsearch.index.query.FilterBuilders
import org.elasticsearch.search.aggregations.AggregationBuilders
import org.elasticsearch.search.aggregations.bucket.histogram.DateHistogram
import org.joda.time.format.DateTimeFormat
import org.joda.time.{DateTime, Duration, Interval}
import example.IntervalsQueryActor.{QueryResults, StartQuery}
import example.RequestExecutor

object IntervalsQueryActor {
    def props(esclient: Client, index: String, types: Seq[String],
              start: DateTime, duration: Duration): Props =
        Props(new IntervalsQueryActor(esclient, index, types, start, duration))

    case class StartQuery()
    case class QueryResults(intervals: Seq[Interval])
}

class IntervalsQueryActor(esclient: Client, index: String, types: Seq[String], start: DateTime, duration: Duration)
        extends Actor with ActorLogging {

    val basicDateTimeFormat = DateTimeFormat.forPattern("yyyyMMdd'T'HHmmss.SSSZ")

    import context.dispatcher

    def receive = LoggingReceive {
        case q: StartQuery =>
            queryIntervals() map (QueryResults(_)) recover {
                case ex =>
                    log.error(ex, s"Retrieving intervals failed")
            } pipeTo sender()
    }

    def queryIntervals(): Future[Seq[Interval]] = {
        val agg = AggregationBuilders.dateHistogram("intervals")
            .field("timestamp")
            .format("basic_date_time")
            .interval(DateHistogram.Interval.seconds(duration.getStandardSeconds.toInt))

        val req = esclient.prepareSearch(index)
            .setSearchType(SearchType.COUNT)
            .setTypes(types : _*)
            .setPostFilter(FilterBuilders.rangeFilter("timestamp").gte(start))
            .addAggregation(agg)

        RequestExecutor[SearchResponse]().execute(req).map { response =>
            val agg = response.getAggregations.get("intervals").asInstanceOf[DateHistogram]
            import scala.collection.JavaConversions._
            val intervals = agg.getBuckets.filter(_.getDocCount > 0)
                .map(bucket => IntervalScanner.basicDateTimeFormat.parseDateTime(bucket.getKey)).toSeq
            intervals.dropRight(1).map(new Interval(_, duration))
        }
    }
}
{% endhighlight %}

# Testing

Usually integration tests involving databases are not so simple as it is the case
with unit tests, but the fact that we can create an embedded instance of Elasticsearch
makes it very convenient and easy to use.

You can encapsulate all the code required to create an embedded node with the
following code (adapted from [this
post](http://cupofjava.de/blog/2012/11/27/embedded-elasticsearch-server-for-tests/)):

{% highlight scala %}
package example

import java.io.IOException
import java.nio.file._
import java.nio.file.attribute.BasicFileAttributes

import org.elasticsearch.common.settings.ImmutableSettings
import org.elasticsearch.node.NodeBuilder

object EmbeddedNode {
    val defaultSettings = Map(
            "node.name" -> "Testing",
            "http.enabled" -> "false")

    def apply() = {
        new EmbeddedNode(defaultSettings)
    }
}

class EmbeddedNode(settings: Map[String, String]) {

    val settingsWithDefaults = (settings.keySet ++ EmbeddedNode.defaultSettings.keySet).map { key =>
        key -> settings.getOrElse(key, EmbeddedNode.defaultSettings(key))
    } toMap

    val dataPath = settingsWithDefaults.getOrElse("path.data",
        Files.createTempDirectory("temp-").toString)

    import scala.collection.JavaConversions.mapAsJavaMap
    val settingsBuilder = ImmutableSettings.builder().put(mapAsJavaMap(settingsWithDefaults))

    if (!settingsWithDefaults.contains("path.data"))
        settingsBuilder.put("path.data", dataPath)

    private lazy val node = NodeBuilder.nodeBuilder()
        .local(true)
        .settings(settingsBuilder.build())
        .build()

    def client() = {
        node.client()
    }

    def start() = {
        node.start()
        this
    }

    def createAndWaitForIndex(index: String): Unit = {
        client.admin.indices.prepareCreate(index).execute.actionGet()
        client.admin.cluster.prepareHealth(index).setWaitForActiveShards(1).execute.actionGet()
    }

    def shutdown(delete: Boolean = true) = {
        node.stop()
        node.close()
        if (delete)
            deleteData
    }

    def deleteData() = {
        try {
            Files.walkFileTree(Paths.get(dataPath), new SimpleFileVisitor[Path] {
                override def visitFile(file: Path, attrs: BasicFileAttributes): FileVisitResult = {
                    Files.deleteIfExists(file)
                    super.visitFile(file, attrs)
                }
                override def postVisitDirectory(dir: Path, exc: IOException): FileVisitResult = {
                    Files.deleteIfExists(dir)
                    super.postVisitDirectory(dir, exc)
                }
            })
        } catch {
            case e: IOException =>
                throw new RuntimeException("Could not delete data directory of embedded elasticsearch server", e);
        }
    }
}
{% endhighlight %}

Now, we are ready to see a test example using [TestKit](http://doc.akka.io/docs/akka/snapshot/scala/testing.html)
 and [ScalaTest](http://www.scalatest.org/).

{% highlight scala %}
package example

import scala.concurrent.duration.DurationInt
import akka.actor.{Actor, ActorSystem}
import akka.testkit.{ImplicitSender, TestActorRef, TestKit}
import org.elasticsearch.client.Client
import org.elasticsearch.common.xcontent.XContentFactory
import org.joda.time.format.DateTimeFormat
import org.joda.time.{Duration, Interval}
import org.scalatest._
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.time.{Milliseconds, Span}
import example.IntervalsQueryActor.{QueryResults, StartQuery}
import example.EmbeddedNode

class IntervalsQueryActorTest(_system: ActorSystem) extends TestKit(_system) with ImplicitSender
        with MustMatchers with FlatSpecLike with ScalaFutures with BeforeAndAfterAll {

    def this() = this(ActorSystem("testing"))

    val node: EmbeddedNode = EmbeddedNode()

    val dateFmt = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss")
    val startDate = dateFmt.parseDateTime("2015-05-03 00:00:00")
    val duration = new Duration(1.second.toMillis)

    val expectedIntervals = List(
        new Interval(dateFmt.parseDateTime("2015-05-03 00:00:00"), duration),
        new Interval(dateFmt.parseDateTime("2015-05-03 00:00:01"), duration),
        new Interval(dateFmt.parseDateTime("2015-05-03 00:00:02"), duration),
        new Interval(dateFmt.parseDateTime("2015-05-03 00:00:03"), duration)
    )

    override def beforeAll {
        node.start()
        node.createAndWaitForIndex("test")
        indexDocuments(node.client())
    }

    override def afterAll {
        TestKit.shutdownActorSystem(system)
        node.shutdown(true)
    }

    def indexDocuments(client: Client) = {
        indexTimestamp(client, "2015-05-03 00:00:00")
        indexTimestamp(client, "2015-05-03 00:00:01")
        indexTimestamp(client, "2015-05-03 00:00:02")
        indexTimestamp(client, "2015-05-03 00:00:03")
        indexTimestamp(client, "2015-05-03 00:00:04")
    }

    def indexTimestamp(client: Client, ts: String) = {
        client.prepareIndex("test", "test")
            .setRefresh(true)
            .setSource(XContentFactory.jsonBuilder().startObject()
            .field("timestamp", dateFmt.parseDateTime(ts).toString())
            .endObject())
            .execute().actionGet()
    }

    "An IntervalsQueryActor" should "be able to query for intervals" in {

        val client = node.client()
        val actorRef = TestActorRef(new IntervalsQueryActor(client, "test", List("test"), startDate, duration))

        val intervalsFuture = actorRef.underlyingActor.queryIntervals()
        whenReady(intervalsFuture, timeout(Span(300, Milliseconds))) { intervals =>
            intervals.size mustBe expectedIntervals.size
            intervals mustEqual expectedIntervals
        }
    }

    it should "return a QueryResults message when receives a StartQuery" in {

        val client = node.client()
        val actorRef = system.actorOf(IntervalsQueryActor.props(client, "test", List("test"), startDate, duration))

        actorRef ! StartQuery()
        expectMsg(500 milliseconds, QueryResults(expectedIntervals))
    }
}
{% endhighlight %}

The first test is an example of how to test an actor feature accessing directly
to the underlying actor class. Using _TestActorRef_ in this way any message
sent to the actor is processed in the same thread. In the second example the
actor is created using the actor props with the actor system and the messages
are processed in other thread. For further details on how to test Akka actors
you can read the [Akka documentation](http://doc.akka.io/docs/akka/snapshot/scala/testing.html).

# Summary

We have seen very briefly how to use the Elasticsearch native Java client from
Scala using Futures, how to launch an embedded server for testing and how to
write an integration test for an Akka actor that uses an Elasticsearch database.
