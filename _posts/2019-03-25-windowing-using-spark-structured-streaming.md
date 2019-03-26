---
title: Windowing Kafka Streams using Spark Structured Streaming
description: What offers Spark Structured Streaming compared to its predecessor Spark
  Streaming. How to read JSON content from a Kafka Stream. How to aggregate data using
  spark windowing and watermarking.
featured: images/spark_streaming.png
layout: post
---

One of the most recurring problems that streaming solves is how to aggregate data during different periods of time. In a previous [post](https://dvirgiln.github.io/akka-streams-windowing/) we showed how the windowing technique can be utilised using Akka Streams. The goal of this post is to show how easy can be done windowing using Spark. 

In my experience all the companies have the same use case to solve: streaming data is coming from a data source, for example Kafka, and it is required to do some transformations over the data to generate a useful dataset for analysis usage. This is commonly known as ETLs (Extract, Transform and Load). In ETLs is quite common to do **aggregations** of data, for example total value of one column, average, count...

All of these operations are provided by Spark, so you do not need to implement them. On the other hand, in Akka Streams all of these operations have to be implemented, as it is a lower level purpose library.

In multiple available online posts you can find examples about how to read and write in kafka and how to use Spark Structured Streaming API, but you won't find a good example. neither in the official documentation nor any other page about how to include multiple aggregations in the same window. We will show you how to do it.

Spark has evolved a lot from its inception. Initially the streaming was implemented using DStreams. From Spark 2.0 it was substituted by Spark Structured Streaming.  Let's take a quick look about what Spark Structured Streaming has to offer compared with its predecessor.

## Differences between DStreams and Spark Structured Streaming

Spark Structured Streaming is the evolution of DStreams. So what are the differences:


### RDDs vs Dataframes/Datasets
In DStreams the data is stored as RDDs while Spark Structured streaming uses Dataframes/Datasets.  
In one side RDDs are more flexible and allow to do much more low level operations, but on the other hand Datasets/Dataframes offer the usage of Spark SQL and they are great for the 90 per cent of the cases. 
Dataframes use the tree based Catalyst SQL query optimizer that improves significantly the Spark performance in terms of speed and memory. 
On the other hand Datasets provide type safety, as all of our queries would be done with JVM objects. We can consider a Dataframe as a Dataset[Row].

### Real Streaming
DStreams store the data into microbatches that simulate real time processing, while Spark Structured Streaming appends the real time events to the processing flow.
DStreams microbatches are always executed even if there is no new data flowing to the stream while in Structured Streaming there is a dedicated thread that checks if there is new data flowing to the stream. If no data is available then the stream query is not executed. This is a significant difference between Spark Streaming (DStreams) and Spark Structured Streaming.

### Windowing Event time
Both DStreams and Structured Streaming provide grouping by windows, but with DStreams it is not possible to include the event time from one of the columns of the incoming data. 
In Structured Streaming it is possible to make windows specifing the window period, the slide lengh and the event time column. 

### Sinks
Using DStreams the output of the streaming is an RDD that can be manipulated. There is not any requirement to use any sink as output. 

Using Structured Streaming requires the usage of an output sink. The output can be Hive, Parquet, Console... Since Spark 2.4 it is posisble to output the streaming computation result into a Dataframe using the **foreachBatch** sink.

## Working Example
 
The example will show different basic aspects of Spark Structured Streaming:
* How to read from a Kafka topic.
* How to deserialize the Json value of the Kafka Stream.
* How to create stream windows.
* Proves how the watermark works.

The first thing to create a streaming app is to create a **SparkSession**:
```
    import org.apache.spark.sql.SparkSession

    val spark = SparkSession
      .builder
      .appName("StructuredConsumerWindowing")
      .getOrCreate()
```

To avoid all the INFO logs from Spark appearing into the Console,  just set the log level as ERROR:
```
spark.sparkContext.setLogLevel("ERROR")
```

Then we need to define our input stream:

```
    val inputStream = spark
      .readStream.format("kafka")
      .option("kafka.bootstrap.servers", kafkaEndpoint)
      .option("auto.offset.reset", "latest")
      .option("value.deserializer", "StringDeserializer")
      .option("subscribe", "shops_records")
      .load
    inputStream.printSchema()
```

The schema from the stream dataframe is:

```
| root
|  |-- key: binary (nullable = true)
|  |-- value: binary (nullable = true)
|  |-- topic: string (nullable = true)
|  |-- partition: integer (nullable = true)
|  |-- offset: long (nullable = true)
|  |-- timestamp: timestamp (nullable = true)
|  |-- timestampType: integer (nullable = true)
```
The Kafka input stream schema is always the same, and it cannot be changed when defining your dataframe.

The records read from the Kafka topic have a json structure based on this case class:

```
SalesRecord(transactionTimestamp: String, shopId: Int, productId: Int, amount: Int, totalCost: Double)
```

So we need to convert our Kafka topic input stream value, that as you see above it has a binary type, into a meaningful dataframe:

```
    val schema = StructType(
      List(
        StructField("transactionTimestamp", TimestampType, true),
        StructField("shopId", IntegerType, true),
        StructField("productId", IntegerType, true),
        StructField("amount", IntegerType, true),
        StructField("totalCost", DoubleType, true)
      )
    )
		 val initial = inputStream.selectExpr("CAST(value AS STRING)").toDF("value")
    initial.printSchema()

```

In this case we selected the value, as we are not interested in the other values provided by the kafka stream. This is the output of the schema:

```
|  |-- value: string (nullable = true)
```

As you noticed this is not exactly what we want. We want to convert this String, to its JSON representation. Let's do it:

```
val aggregation = initial.select(from_json($"value", schema)
aggregation.printSchema()
```

With the previous expression the input stream is being deserialized to its JSON value. This is how the schema looks like:

```
| root
|  |-- jsontostructs(value): struct (nullable = true)
|  |    |-- transactionTimestamp: timestamp (nullable = true)
|  |    |-- shopId: integer (nullable = true)
|  |    |-- productId: integer (nullable = true)
|  |    |-- amount: integer (nullable = true)
|  |    |-- totalCost: double (nullable = true)
```

As you can notice, there is a nested structure **jsontostructs** that contains all the json fields. We need to select the embedded values:

```
    val aggregation = initial.select(from_json($"value", schema).alias("tmp")).select("tmp.*")
    aggregation.printSchema()
```

With the **.select("tmp.*")** we are selecting the embedded content. This is the final value of the dataframe schema:
```
| root
|  |-- transactionTimestamp: timestamp (nullable = true)
|  |-- shopId: integer (nullable = true)
|  |-- productId: integer (nullable = true)
|  |-- amount: integer (nullable = true)
|  |-- totalCost: double (nullable = true)
```

Looking good. Lets progress.

We now want to define the **window** size and **watermark**:

```
def window(timeColumn: Column, windowDuration: String, slideDuration: String): Column 
```
The window has 3 parameters:
* timeColumn: this is one of the key differences with DStreams. You can define your windows based on the event timestamp column. Nice.
* windowDuration: defines the window size.
* slideDuration: defines how the windows are moving. 

If the slideDuration is less than the window duration it means we would have **overlapping windows**. In our example we do not want overlapping windows. We want that every SalesRecord just belongs to one window at a time. That's why we will set the same value for the windowDuration and the slideDuration.

This is the code that shows how to define the window and watermark:

```
    val windows = aggregation
      .withWatermark("transactionTimestamp", "5 minutes")
      .groupBy(window($"transactionTimestamp", "1 minute", "1 minute"), $"shopId")
```
First it has been defined a watermark of 5 minutes. That means that the window would be open waiting for 5 minutes for late records.
To define a window, it is required to do a **groupBy** operation.  In our case we are grouping by window and shopId.

The output of the **groupBy** operation is not a dataframe. It is a RelationalGroupedDataset. The operations allowed by this class are: avg, count, agg and pivot. 

When you execute the operation avg or count, it generates a Dataframe with the grouped columns plus an aditional column avg or count. In our case we want a dataframe with multiple aggregations. To do that it is required to use the **agg** operation:

```
    import org.apache.spark.sql.functions._
   val aggregatedDF = windows.agg(sum("totalCost"), count("*"))
```
It is quite easy to include multiple aggregations to the result dataframe. The only requirement is to include the import of the default functions provided by spark. Take a look to this class to see all the functions you can use in your aggregations.

The final step is writing the aggregated data into a sink. 
```
    val dfcount = aggregatedDF.writeStream.outputMode("complete").option("truncate", false).format("console").start()
    dfcount.awaitTermination()
```

In our case the sink used is the console, but it could have been hive, another Kafka topic, parquet...

It is important to notice the parameter outputMode. We will go more in detail in the next post.

The console output is:

```
| Batch: 19
| -------------------------------------------
| +------------------------------------------+------+------------------+--------+
| |window                                    |shopId|sum(totalCost)    |count(1)|
| +------------------------------------------+------+------------------+--------+
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|2     |657.8800000000001 |24      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|2     |1758.8000000000002|55      |
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|1     |790.95            |26      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|1     |1645.8400000000001|45      |
| +------------------------------------------+------+------------------+--------+
|
| -------------------------------------------
| Batch: 20
| -------------------------------------------
| +------------------------------------------+------+------------------+--------+
| |window                                    |shopId|sum(totalCost)    |count(1)|
| +------------------------------------------+------+------------------+--------+
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|2     |753.8800000000001 |27      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|2     |1758.8000000000002|55      |
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|1     |974.9200000000001 |33      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|1     |1645.8400000000001|45      |
| +------------------------------------------+------+------------------+--------+
|
| -------------------------------------------
| Batch: 21
| -------------------------------------------
| +------------------------------------------+------+------------------+--------+
| |window                                    |shopId|sum(totalCost)    |count(1)|
| +------------------------------------------+------+------------------+--------+
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|2     |843.8700000000001 |29      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|2     |1758.8000000000002|55      |
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|1     |1138.8700000000001|41      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|1     |1645.8400000000001|45      |
```

## Source Code
All the source code can be found in my github account:

[https://github.com/dvirgiln/spark-windowing](https://github.com/dvirgiln/spark-windowing)

All the problem has been dockerized. You just need to follow these instructions:

```
    1. sbt docker
    2. docker swarm init
    3. docker stack deploy -c docker-compose.yml spark-windowing
    4. docker service ls
    5. docker service logs -f spark-windowing_producer
    6. docker service logs -f spark-windowing_spark-consumer
    7. docker stack rm spark-windowing
    8. docker swarm leave --force
```

## Conclusion
This article has been very dense and it shows from the beginning how to read a Kafka Stream and use the powerful features provided from Spark: window and watermark. Apart from that it shows how to deserialize the json content and make multiple aggregations in the same window.

My initial idea was to include as well examples that proof how the different output modes and watermark work, but as the lenght of the post exceeded my initial idea, i will talk about them in another article.