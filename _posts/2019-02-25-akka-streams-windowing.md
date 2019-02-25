---
title: Windowing using Akka Streams and Scala
featured: images/akka.png
layout: post
description: Good introduction to Akka Streams explaining what it is windowing and
  how to implement it using Scala and Akka Streams.
---

Akka Streams is a powerful streaming library that allows developers to manipulate the data in multiple and genuine ways, but it is not providing aggregations of the data like you can find in other Streaming libraries like Spark or Kafka Streams. 

Aggregating data is one of the most common problems that streaming applications require. Akka streams is a low level library that allows developers to do great stuff and manipulate the incoming data in such amazing ways. The main difference between Akka Streams and Apache Spark is that Spark is a higher level library that includes more boundaries and limitates what the developer can do. 

I would like to explain more about the differences between Akka Streams and Spark Streaming, but that is not part of this article. What we need to know is that Spark Streaming, as a higher level library provides Windowing functionality while Akka Streams doesn't.

But, what is it exactly Windowing?

## Windowing: the concept
The data that is consumed from the stream needs to be aggregated by a different window of time. We could have aggregations of the data by windows of 5 minutes, or windows of 30 minutes. For example we could have windows of 30 minutes, so, it starts from 00:00 -00:30, 00:30 - 01:00, 01:00 - 01:30 ,,,
Some examples about the aggregation of the data could be:
* Bookings: aggregate the gross cost of all the online shop bookings in windows of 5 minutes. This data could be ingested to a machine learning pipeline looking for anomalies.
* Music platform: aggregate the songs listened per region and per artist in windows of 5 minutes. This information can be used to make suggestion to other users.

But in this case the first question is, *"Ok but, how I get the event time of a record consumed from the stream?"* . There are two ways:
* **Event time** included in the message: the records consumed contain an attribute that specifies when the record happened. For instance in the booking example it could be the transaction_timestamp.
* **Processing time**: the event time is got from the system timestamp of the akka streams application.

The processing time based windows is not exactly what we want, as the incoming records are comming unordered. It could happen that we are processing a record that happened 5 minutes ago, so using the processing time as mechanism to assing a record to a window is not ideal. 

We want to assign our records to different windows of time based on a record attribute that will tell us to which window it belongs to.

Then, we need to decide if we want to have **sliding windows** or **tumbling windows**:
* *Sliding Windows*: the windows overlap one with each other depending of the step time.  One record can be part of different windows at the same time. For instance we can have a window with lenght 30 minutes and steps of 10 minutes, these would means for a time based starting from 00:00: W1(00:00 -00:30), W2(00:10 - 00:40), W3 (00:20 - 00:50) ...
* *Tumbling Windows*: there is no overlap. Every record belongs just to one window. For instance for a window with lenght of 30 minutes, the windows associated started from 00:00 would be: W1(00:00 - 00:30), W2(00:30,01:00), W3(01:00 - 01:30) ...

There is another concept we need to speak about when it comes to streams windows before we start the real implementation: **watermark**.

As we commented briefly before, the records they do not come ordered. Watermarking is a techique that deals with lateness. It is the threshold that tell the system how long it has to wait waiting for late records. 

So, we need to implement a mechanism based on windows that aggregates the incoming records into the window/s it belongs to. There are 3 parameters that defines the windows.
* Window lenght;
* Window step:
* Watermark

As well it is require to define:
* Aggregate result type: every time a new record is added to the window, it aggregates the record to the aggregation result.
* Window Status type: OpenWindow and ClosedWindow.
*