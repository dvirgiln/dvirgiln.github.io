---
title: Windowing using Akka Streams and Scala
featured: images/akka.png
layout: post
description: Good introduction to Akka Streams explaining what it is windowing and
  how to implement it using Scala and Akka Streams.
---

Akka Streams is a powerful streaming library that allows developers to manipulate streaming data in multiple and genuine ways. In the other hand, it is not providing aggregations of the data like you can find in other Streaming libraries like Spark or Kafka Streams.

Aggregating data is one of the most common problems that streaming applications require. Akka streams is a low level library that allows developers to do great stuff and manipulate the incoming data in such amazing ways. The main difference between Akka Streams and **Apache Spark** is that Spark is a higher level library that includes more boundaries and limitates what the developer can do.

I would like to explain more about the differences between Akka Streams and Spark Streaming, but that is not part of this article. What we need to know is that Spark Streaming, as a higher level library provides Windowing functionality while Akka Streams doesn't.

But, what is it exactly streaming windowing?

## Windowing: the concept
The data that is consumed from the stream needs to be aggregated by different windows of time. We could have aggregations of the data by windows of 5 minutes, or windows of 30 minutes. For example we could have windows of 30 minutes, so, it starts from 00:00 -00:30, 00:30 - 01:00, 01:00 - 01:30 ,,,
Some examples about data aggregation could be:
* Bookings: aggregate the gross cost of all the online shop bookings in windows of 5 minutes. This data could be ingested to a machine learning pipeline looking for anomalies.
* Music platform: aggregate  songs listened per region and per artist in windows of 5 minutes. This information can be used to make suggestion to other users.

But in this case, the first question is, *"Ok but, how I get the event time of a record consumed from the stream?"* . There are two ways:
* **Event time** included in the message: the records consumed contain an attribute that specifies when the record happened. For instance in the booking example it could be the transaction_timestamp.
* **Processing time**: the event time is got from the system timestamp of the akka streams application.

The processing time based windows is not exactly what we want, as the incoming records are comming unordered. It could happen that we are processing a record that happened 5 minutes ago, so using the processing time as mechanism to assing a record to a window is not ideal.

We want to assign our records to different windows of time based on a record attribute that will tell us to which window they belong to.

Then, we need to decide if we want to have **sliding windows** or **tumbling windows**:
* *Sliding Windows*: the windows overlap one to each other depending of the **step time**.  One record can be part of different windows at the same time. For instance we can have a window with lenght 30 minutes and steps of 10 minutes, these would means for a time based starting from 00:00 our windows would be: W1(00:00 -00:30), W2(00:10 - 00:40), W3 (00:20 - 00:50) ...
* *Tumbling Windows*: there is no overlap. Every record belongs just to one window. For instance for a window with lenght of 30 minutes, the windows associated started from 00:00 would be: W1(00:00 - 00:30), W2(00:30,01:00), W3(01:00 - 01:30) ...

There is another concept we need to speak about when it comes to streams windows before we start the real implementation: **watermark**.

As we commented briefly before, the records do not come ordered. Watermarking is a techique that deals with lateness. It is the threshold that tells the system how long it has to wait waiting for late records.

So, we need to implement a mechanism based on windows that aggregates the incoming records into the window/s it belongs to. There are 3 parameters that defines the windows.
* Window lenght
* Window step
* Watermark

## Implementation
In the following example we will go throw a shops booking stream. The stream will receive booking records from different shops, specifying the total amount of items and the cost.

In our case the SalesRecord case class looks like this:

```
  case class SalesRecord(transactionTimestamp: Long, shopId: Int, productId: Int, amount: Int, totalCost: Double)
```

As you can notice the record itself contains contains the event time. Good!

We want to implement a window solution with the following features:
* Tumbling windows: this means that the window step and window lenght is the same value. There will be no overlap between windows. Every record would be part of the only one window.
* Watermark: 5 seconds.

In our example our Akka Streams App is consuming records from Kafka. This is how we define the Kafka Consumer:

```
  val consumerSettings = ConsumerSettings(system, new ByteArrayDeserializer, new ByteArrayDeserializer)
    .withBootstrapServers(kafkaEndpoint)
    .withGroupId("group1")
    .withProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
	val source = Consumer.plainSource(consumerSettings, Subscriptions.topics("shops_records")).mapAsync(1) { msg =>
    val value = deserialise(msg.value).asInstanceOf[SalesRecord]
    Future(value)
  }
```

It is deseralizing the Kafka record into a SalesRecord object. Easy.

Before we continue, let's recap quickly the different components of a Akka Stream:
* **Source**: a processing stage that contains exatly one output. This is the beginning of the streaming app.
* **Flow**: a processing stage that contains one input and one output. This is a processing middle stage.
* **Sink**: a processing stage that contains exactly one input. This is the last part of the processing. There could be a FileSink, an ActorSink..., or you could define your own Sink.


After this quick recap, let's focus in our example. Now it is coming the complicated part: creating the windows and assigning events to them. In our case we want one a tumbling window, so it means that the windows do not overlap with each other.

This is how the Window case class looks like:

```
case class Window(from: Long, to: Long, duration: Option[Long] = None) {
  override def toString() = s"From ${ConsumerUtils.tsToString(from)} to ${ConsumerUtils.tsToString(to)}"
}
```

Then we want to convert our Streaming of SalesRecords into a streaming of **window commands**. This means that for every record in time we need to generate a set of commands that would open new windows, close existing windows and add a record to a window. You will see in a few minutes how to generate this commands, and all the logic behind. For the moment this is how the window commands look like:

```
sealed trait WindowCommand {
  def w: Window
}

case class OpenWindow(w: Window) extends WindowCommand
case class CloseWindow(w: Window) extends WindowCommand
case class AddToWindow(ev: SalesRecord, w: Window) extends WindowCommand
```

In the previous code we model the three different commands that define the different actions to do in the stream. 

The following code it is the starting point of the streaming processing:
```
  //Generates window commands
  val subFlow = source.statefulMapConcat { () =>
    logger.info(s"Frequencies: $configs")
    val generator = new WindowCommandGenerator(configs.map(_.frequency))
    event =>
      val timestamp = event.transactionTimestamp
      generator.forEvent(timestamp, event)
  }.groupBy(64, command => command.w)
```
Considerations about the previous code:

* Line 1: the syntax of *def statefulMapConcat[T](f: () ⇒ Out ⇒ immutable.Iterable[T])*. It is used to convert a single record into an Iterable. In our case every record will generate a collection of WindowCommands.
* Line 3: It is created an instance of the WindowCommandGenerator. This is key part and we will explain it afterwards. It receives a sequence of frequencies and then generate window commands for all the frequencies. For instance, we maybe for the same stream would like to aggregate the data in two different windows, for instance 5 and 30 minutes windows.
* Line 4: For every record that we receive then...
* Line 6: the generator will generate for that timestamp a collection of window commands: Open, Close and Add to Window commands.
* Line 7: the **groupBy** operation creates up to 64 **substreams** having as a key the window. So all the WindowCommands that belongs to the same window will be part of the same substream. GroupBy returns a special Stage type. It is not a Flow or Source, it is a **SubFlow**.

The following code is a subpart of the WindoCommand Generator. It is the function that returns a set of windows for a timestamp:

```
import scala.concurrent.duration._


object Window {
  def windowsFor(ts: Long, duration: FiniteDuration): Set[Window] = {
    //As the window length and window step is one, it will return just a Set of one window.
    val WindowLength = duration.toMillis
    val WindowStep = duration.toMillis
    val WindowsPerEvent = (WindowLength / WindowStep).toInt

    val firstWindowStart = ts - ts % WindowStep - WindowLength + WindowStep
    (for (i <- 0 until WindowsPerEvent) yield Window(
      firstWindowStart + i * WindowStep,
      firstWindowStart + i * WindowStep + WindowLength, Some(duration.toMillis)
    )).toSet
  }
}
```
In our case, we want tumbling windows, so they do not overlap one to each other, so, in this case the **window step** has the same duration as the window. So in our case calling to this method will return just one window. In case you want to have **sliding windows**, you need to adjust the window step duration.

The following code shows the implementation of the Window Command Generator. Do not be afraid. We will go over it:

```
class WindowCommandGenerator(frequencies: Seq[FiniteDuration]) {
  private val MaxDelay = 5.seconds.toMillis
  private var watermark = 0L
  private val openWindows = mutable.Set[Window]()

  def forEvent(timestamp: Long, event: SalesRecord): List[WindowCommand] = {
    watermark = math.max(watermark, timestamp - MaxDelay)
    if (timestamp < watermark) {
      println(s"Dropping event ${event} with timestamp: ${ConsumerUtils.tsToString(timestamp)}")
      Nil
    } else {
      val closeCommands = openWindows.flatMap { ow =>
        if (ow.to < watermark) {
          openWindows.remove(ow)
          Some(CloseWindow(ow))
        } else {
          None
        }
      }
      //This is key. It checks the different frequencies and it filters out the frequencies that already have an open window.
      val toBeOpen = frequencies.filter(w => !openWindows.exists(ow => ow.duration.get == w.toMillis))

      val openCommands = toBeOpen.flatMap { duration =>
        //it creates a list of windows with the frequency defined by the config.
        val listWindows = Window.windowsFor(timestamp, duration)
        listWindows.flatMap { w =>
          openWindows.add(w)
          Some(OpenWindow(w))
        }
      }
      val addCommands = openWindows.map(w => AddToWindow(event, w))
      openCommands.toList ++ closeCommands.toList ++ addCommands.toList
    }
  }
}
```

* Line 35:  the function returns a list of commands as the result of the concatenation of the open, close and add commands. We need to see how these 3 list of commands are generated:
    * **Close Window Command**: in the line 13, if the existing openwindows (line 4) are older than the watermark, then remove the openWindow and generate a new CloseWindow command.
    * **Open Window Command**: in the line 23, if a window for one frequency was already closed, then it is required to create an OpenWindow command. In the line 28, we can see the call to **Window.windowsFor** to generate the new windows. Then the new windows then are added to the openWindows list(line 30).
    * **Add to Window command**: from all the open windows, then we need to add the current event to the window (line 34).

Once it has been explained how to generate the windows and how to create a flow of window commands, then we can come back to the original **subflow**:

```
  //Generates window commands
  val subFlow = source.statefulMapConcat { () =>
    logger.info(s"Frequencies: $configs")
    val generator = new WindowCommandGenerator(configs.map(_.frequency))
    event =>
      val timestamp = event.transactionTimestamp
      generator.forEvent(timestamp, event)
  }.groupBy(64, command => command.w)
```

So the result type of this code  is a SubFlow type of WindowCommand. Then we need to aggregate the results of each window:

```
      val aggregator = Flow[WindowCommand].fold(List[SalesRecord]()) {
        case (agg, OpenWindow(window)) => agg
        case (agg, CloseWindow(_)) => agg
        case (agg, AddToWindow(ev, _)) => ev :: agg
      }
```
This aggregator uses the fold operation. So, it aggregates all the SalesRecords into one List.

## Source Code
All the source code is available in my github account:

[https://github.com/dvirgiln/streams-kafka](https://github.com/dvirgiln/streams-kafka)

Go into the Akka Consumer directory. The code explained here it is just a small subpart of this repository. In the next article we will go through the powerful **GraphDSL** functionality provided by Akka Streams.

## Conclusion
This article could be a good starting point to initiate into Akka Streams. We have covered quickly all the required theorical terms to understand what it is and how to implement windowing using Akka Streams.

And the most important thing, the code explained in this article works! You just need to clone my repo and run it with docker. If you see something in my repository that you do not understand and has not been covered in this post, do not worry. We will go through it in future posts.

I hope you had so much fun, as I had while writing it.