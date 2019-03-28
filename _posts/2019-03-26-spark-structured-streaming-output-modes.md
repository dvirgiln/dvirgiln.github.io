---
title: Spark Structured Streaming output mode.
description: We will explain the Spark Structured Streaming output mode and watermark features
  with a practical exercise based on Docker.
featured: images/spark_streaming.png
layout: post
---

This article is a follow up of our previous [post](https://dvirgiln.github.io/windowing-using-spark-structured-streaming/). In our previous post we explained the differences between Spark Structured Streaming and Spark Streaming and created a Docker example of how to consume data from Kafka using windows and watermark.

Following our previous example, in this article we want to explain one of the key features of Spark Structured Streaming: the output modes. 

We also want to explain how Spark manages late records using watermark.

## Output Modes
The output mode is a new concept introduced by Structured Streaming. As we already mentioned in our previous [post](https://dvirgiln.github.io/windowing-using-spark-structured-streaming/), Spark Structured Streaming requires a sink. 

The output mode specifies the way the data is written to the result table. These are the three different values:

 * **Append** mode: this is the default mode. Just the new rows are written to the sink. 
 * **Complete** mode: it writes all the rows. It is supported just by groupBy or groupByKey aggregations.
 * **Update** mode: writes in the sink "only" all the rows that are updated.  When there are no aggregations it works exactly the same as "append" mode

Now we are going to see these 3 different append modes in a real problem.

Following our previous post, we were able to aggregate our Sales data in a dataframe that looked like this:
```
| +------------------------------------------+------+------------------+--------+
| |window                                    |shopId|sum(totalCost)    |count(1)|
| +------------------------------------------+------+------------------+--------+
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|2     |843.8700000000001 |29      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|2     |1758.8000000000002|55      |
| |[2019-03-26 09:02:00, 2019-03-26 09:03:00]|1     |1138.8700000000001|41      |
| |[2019-03-26 09:01:00, 2019-03-26 09:02:00]|1     |1645.8400000000001|45      |
```

Let's explore how the data is written to the sink depending the different output modes chosen:

### Complete Output Mode
This is the console output generated when the output mode is set to **complete**:
```
| -------------------------------------------
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

As you will notice by the different batches, all of the windows are being written to the sink, even though some of the windows have not been updated during that batch. This is the key feature of **complete** mode. With every record update, everything has been written again, not just the updated rows.

### Update Output Mode
This is the console output generated when the output mode is set to **update**:
```
| -------------------------------------------
| Batch: 3
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:52:00, 2019-03-26 08:53:00]|1     |120.0         |2       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 4
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:53:00, 2019-03-26 08:54:00]|2     |40.0          |1       |
| +------------------------------------------+------+--------------+--------+
|
```
In this case, every batch writes just the rows that have been updated. This output mode is normally what makes more sense in most cases.

## Watermark
Watermark is the streaming property that allows the stream to wait for late records. The records that have been streamed after the watermark has expired are discarded.

Watermark makes sense when it is defined on a stream grouped by windows. So windows stay open waiting for late records, until the watermark has expired for those windows.

To test how watermark works, we'll use **update** output mode.

We want to reproduce what normally happens in production ETLs: there is a stream of data that is aggregated in a window period, but there are some late records that come with a delay. So the real time streaming job will follow these steps:
* Aggregate the data that is streamed to every window.
* Push the aggregated data into the sink.
* If a new late record comes before the watermark expires, it has to be aggregated and written to the sink.


We are going to test this use case in our example:


1. Define the window size and watermark:
```
    val windows = aggregation                                                        
      .withWatermark("transactionTimestamp", "5 minutes")                            
      .groupBy(window($"transactionTimestamp", "1 minute", "1 minute"), $"shopId")   
```
As we can see, there are windows with a time of one minute and a watermark of 5 minutes. So we are grouping all the data that is coming every minute and wait 5 minutes for late records.

2. The producer sends 2 records every minute.
```
 - Producing record: SalesRecord(2019-03-26 08:50:30.174,1,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:50:30.174,1,3,3,45.0)  
 - Producing record: SalesRecord(2019-03-26 08:51:30.174,1,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:51:30.174,1,4,1,22.0)  
 - Producing record: SalesRecord(2019-03-26 08:52:30.174,1,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:52:30.174,1,2,2,80.0)  
 - Producing record: SalesRecord(2019-03-26 08:53:30.174,2,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:53:30.175,2,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:54:30.174,2,3,1,15.0)  
 - Producing record: SalesRecord(2019-03-26 08:54:30.174,1,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:55:30.175,1,2,1,40.0)  
 - Producing record: SalesRecord(2019-03-26 08:55:30.175,1,1,2,19.98) 
 - Producing a late Record...                                         
 - Late Record...                                                     
 - Producing record: SalesRecord(2019-03-26 08:51:30.174,1,4,2,44.0)  
 - Producing record: SalesRecord(2019-03-26 08:57:30.175,1,2,2,80.0)  
 - Producing record: SalesRecord(2019-03-26 08:57:30.175,1,4,1,22.0)  
```
As we can see after 6 minutes it is sent a late record.

3. Check the consumer console output:
```
| -------------------------------------------
| Batch: 1
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:50:00, 2019-03-26 08:51:00]|1     |85.0          |2       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 2
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:51:00, 2019-03-26 08:52:00]|1     |62.0          |2       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 3
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:52:00, 2019-03-26 08:53:00]|1     |120.0         |2       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 4
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:53:00, 2019-03-26 08:54:00]|2     |40.0          |1       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 5
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:53:00, 2019-03-26 08:54:00]|2     |80.0          |2       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 6
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:54:00, 2019-03-26 08:55:00]|2     |15.0          |1       |
| |[2019-03-26 08:54:00, 2019-03-26 08:55:00]|1     |40.0          |1       |
| +------------------------------------------+------+--------------+--------+
|
| -------------------------------------------
| Batch: 7
| -------------------------------------------
| +------------------------------------------+------+------------------+--------+
| |window                                    |shopId|sum(totalCost)    |count(1)|
| +------------------------------------------+------+------------------+--------+
| |[2019-03-26 08:55:00, 2019-03-26 08:56:00]|1     |59.980000000000004|2       |
| +------------------------------------------+------+------------------+--------+
|
| -------------------------------------------
| Batch: 8
| -------------------------------------------
| +------------------------------------------+------+--------------+--------+
| |window                                    |shopId|sum(totalCost)|count(1)|
| +------------------------------------------+------+--------------+--------+
| |[2019-03-26 08:51:00, 2019-03-26 08:52:00]|1     |106.0         |3       |
| +------------------------------------------+------+--------------+--------+
|
```
As we can see the window "*2019-03-26 08:51:00, 2019-03-26 08:52:00*" was written to the sink in batch number 2 and in batch number 8, as it was a late record. In this case, the same window was updated in the sink, updating the count and total cost values.

So this example demonstrates our production use case. There are some records that are aggregated and written to the sink, when it arrives a late record, it updates the sink row, aggregating the new value.

## Source code
All the source code can be found on my github account:

[https://github.com/dvirgiln/spark-windowing](https://github.com/dvirgiln/spark-windowing)

The whole problem has been dockerized. You just need to follow these instructions:

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
This article explains and demonstrates how to use the watermark and output mode features in Spark Structured Streaming. 
This is a continuation of the [previous article](https://dvirgiln.github.io/windowing-using-spark-structured-streaming/) that hopefully will help better your understanding of how Spark Structured Streaming works and how it can solve real production use cases.