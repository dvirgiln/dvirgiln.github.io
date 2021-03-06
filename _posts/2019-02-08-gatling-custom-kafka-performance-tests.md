---
title: Gatling Kafka performance tests
description: How to stress Kafka with thousands of request per second using a Gatling
  Scala codebase. By default it is not possible without creating a custom Gatling
  Action. Let's do it.
layout: post
featured: images/Gatling-dark-logo.png
---

Gatling is a performance scala library that facilitates running performance tests on your web services/applications. By default Gatling works with HTTP Rest requests. So then, how is it possible to run performance tests in Kafka using Gatling? 

It is possible to create custom Actions in Gatling that specify how to stress your application. This is exactly what we are going to explore in this post. 
  

## Custom Kafka Gatling Action

There is not much documentation about how to create a custom gatling action. But it is possible to implement it. Before starting the implementation, I want to explain the logic of the implementation. There are four variables required to run these performance tests:

* *GATLING_NUMBER_USERS_PER_SECOND* : number of users injected by Gatling per second.
* *GATLING_MAX_DURATION_SECONDS* : maximum duration of our Gatling tests.
* *GATLING_DURATION_SECONDS*:  the duration of our tests whilst Gatling is injecting requests.
* *GATLING_NUMBER_RECORDS_PER_TRANSACTION*: number of transactions sent to Kafka per Gatling request. This allows to increase the parallelism to Kafka without adding parallelism to Gatling.

In this case in our example, the goal of the tests is to stress test kafka using an Avro wrapper to communicate with the  Kafka cluster. The AvroKafkaSender communicates with Kafka using the *kafka_clients*  library. But in your case, you can stress test Kafka or other platform using as a client your own library. This could be a good template to start.

To create a custom Gatling action it is required to mixin the following Gatling traits:

* *ExitableAction*
    * contains a function *def execute(session: Session)* that needs to be implemented. 
    * Every request injected by Gatling creates an instance of this class.
* *ActionBuilder*
    * Builder trait that creates an Action. 
    * Contains a function *def build(ctx: ScenarioContext, next: Action): Action*.
    * Creates an instance of the custom action.
 * *Protocol*
     * The protocol is injected into the test scenario.
     * It contains global instances that can be used from the Gatling action. 
     * Here is where we will instantiate our Avro Kafka client.
 * *ScenarioBuilder*
     * Usage of an Scenario builder to build a new scenario.
     * Injects the Action Builder into the scenario builder. 
 * *Simulation*
     * Injects a Scenario Builder into the simulation.
     * Defines the rate of users per second to the Scenario Builder.
     * Inject a protocol/s.
     * Defines the duration of the tests and the maximum duration.

So, it is a bit of work to be done. Lets start from the Simulation to the Action. 

This is the definition of a simulation:

```
class WriteKafkaListenKafkaHotelAvailabilityRequestSpec extends Simulation{
    /*We obvious the code here to read the numberOfUsersPerSecond, duration and maxDuration, as well as the definition of the protocol*/
    
    setUp(scenario.scenarioBuilder.inject(constantUsersPerSec(numberOfUsersPerSecond).during (duration seconds)).
      protocols(protocol)).maxDuration(max_duration  seconds)
}
```

Easy. Let's move on to the ScenarioBuilder.

```
import io.gatling.core.Predef.{scenario, _}
trait AvroKafkaScenario{
	def scenarioBuilder = {
		val generateUUIDs = (1 to numberOfRecorsPerTransaction).map(_ => UUID.randomUUID.toString).toList
		val generateRecords = (ids: List[String]) => ids.map(id => generateRecord(id))
		val keyFunc = () => generateUUIDs
		val payloadFunc = (ids) => generateRecords(ids)
		val requestBuilder= AvroKafkaRequestBuilder("request")
		val avroKafkaActionBuilder= requestBuilder.send[String, GenericData.Record](keyFunc, payloadFunc)
		scenario(s"Avro Kafka Test").exec(avroKafkaActionBuilder)
	}
}
```
About the previous code:
* Line 8: The code that contains how to create a Gatling Action Builder is encapsulated in a the class AvroKafkaRequestBuilder (below).
* The request builder accepts as parameters the keys and the payload. 
* This request builder code is run just once by Gatling when the scenario is created. So that's the reason why instead of passing as parameters for our builder static random values for the keys and payload, I am passing two functions. If we would have include instead of higher order functions, materialized values, that would means that all the Gatling requests would be identical. 
* This is the power of functional programming. Your functions are values as well, as if you declare an Integer or a String.
* Line 10: The *scenario* function is a helper function that generates an skeleton of a ScenarioBuilder. 
* Line 10: The *exec* function accepts an ActionBuilder. 

This is the code that encapsulates the logic for the Request builder:
```
case class AvroAttributes[K <: List[_], V <: List[_ <: IndexedRecord]](requestName: Expression[String], key: Option[() => Expression[K]], payload: (K) => Expression[V])

case class AvroKafkaRequestBuilder(requestName: Expression[String]) {
  def send[K, V <: IndexedRecord](key: () => Expression[List[K]], payload: (List[K]) => Expression[List[V]]): AvroRequestActionBuilder[K, V] = {
    send(payload, Some(key), checkResult)
  }
  private def send[K, V <: IndexedRecord](payload: (List[K]) => Expression[List[V]], key: Option[() => Expression[List[K]]])= new AvroKafkaRequestActionBuilder(AvroAttributes(requestName, key, payload))
}
```
As you can see, maybe this code looks a bit difficult, but it is just a wrapper that allows using Type Parameters to define a generic RequestBuilder for any kind of List of keys and any kind of List of Records. A few considerations explaining the previous code:
* The type parameters of the class are *[K <: List[_], V <: List[_ <: IndexedRecord]]*. This means that the Key can be a List of anything and the payload is going to be a List of a class that extends from IndexedRecord. IndexedRecord is the parent Avro class. Any Avro message autogenerated class extends from IndexedRecord.
* Adding Type Parameters to the builder is quite important. It add flexibility to the tests and allows for example to create Gatling tests with String as a key and other Gatling performance test with String as key. The same applies to the payload. We could want to have Gatling tests for the Avro message class Booking and as well for the Avro message for Cancellations. 
* Both the key and payload parameters for the function send are higher order functions. Here it is the power of the functional programming.
* We define a case class AvroAttributes that it is the one that is injected to the AvroKafkaRequestActionBuilder.

We are almost there. The next class to define is the custom Protocol. This is injected to the scenario just once. It contains global objects used by our custom gatling action.

```
case class AvroProtocol(bucketName: String = "", topic: String = ""){
  lazy val avroKafkaClient: AvroKafkaClient = getAvroKafkaClient
}
```
We do not need to enter into too much detail here about how to instantiate the avroKafkaClient. In this section is where you have to define your kafka client, or your application client definition.

And this is the code for the ActionBuilder.

```
class AvroRequestActionBuilder[K, V <: IndexedRecord](avroAttributes: AvroAttributes[List[K], List[V]]) extends ActionBuilder {
  override def build(ctx: ScenarioContext, next: Action): Action = {
    import ctx.{protocolComponentsRegistry, coreComponents}
    val avroComponents: AvroComponents = protocolComponentsRegistry.components
    new AvroRequestAction(
      avroAttributes,
      coreComponents,
      avroComponents.protocol,
      next
    )
  }
}

```
The builder is quite simple. It tells Gatling how to create actions. Nothing important to notice here, except that we can import the global protocolComponentsRegistry (line 3). From this registry we are able to inject into our action the AvroProtocol. Remember here that the AvroProtocol contains the global objects that can be used by all the actions.

Until now we have just created the Gatling architecture to deal with our custom avro kafka requests. Here it is the code with the logic for the custom AvroKafkaAction.

```
class AvroRequestAction[K, V <: IndexedRecord](val avroAttributes: AvroAttributes[List[K], List[V]], val coreComponents: CoreComponents, val protocol: AvroProtocol, val next: Action)
  extends ExitableAction with NameGen {
  val statsEngine = coreComponents.statsEngine
  override val name = genName("avroRequest")
  def clock: Clock = coreComponents.clock

  override def execute(session: Session): Unit = recover(session) {
    avroAttributes requestName session flatMap { requestName =>
      val outcome =
        sendRequest(
          requestName,
          avroAttributes,
          session)

      outcome.onFailure(
        errorMessage =>
          statsEngine.reportUnbuildableRequest(session, requestName, errorMessage)
      )
      outcome
    }
  }

  private def sendRequest(requestName: String, avroAttributes: AvroAttributes[List[K], List[V]], session: Session) = {
    val before = System.currentTimeMillis()
    avroAttributes.key.value.map { keyValue =>
      keyValue.apply()(session).map { key =>
        avroAttributes.payload.apply(key)(session) map { payload =>
          payload.map(p => protocol.avroKafkaClient.send(p))
          next ! session
          session.markAsSucceeded
          statsEngine.logResponse(session, "Hitting Kafka", before, System.currentTimeMillis(), OK, None, None)
        }
      }
    }.get.flatMap(a => a)

  }
```

A few considerations about the previous code:
* Line 8: this line is exactly the same as doing *avroAttributes.requestName.apply(session)*. Magic? No, Scala.
* All the attributes for the action reside in the avroAttributes object. Remember that the attributes we have are:
    * RequestName
    * Key: function that returns a list of Strings.
    * Payload: function that from a List of Strings returns a list of Avro records.
* Line 23: the main action logic resides here.
* Line 25 and 26: get the list of keys from the attributes object and call the method using *apply*. In case we would have just set the key as a List of String, then we wouldn't be able to generate a different set of keys in every action. We would have been using the same set of keys in all the Gatling request actions.
* Line 27: get the payload from the session and apply the list of keys (remember that the payload attribute is a high order function).
* Line 28: we have the payload as a list of records. Iterates over them and send the request to Kafka. 
* Line 29 to 31: mark the request as successful in Gatling and log the response stats. This helps Gatling to create the stats about average request time, maximum request time...


    
## Conclusion

Create a custom Gatling action is not an easy task. It requires  understanding of how the Gatling code works. In our case we have created a custom Action that sends a list of Avro records to Kafka in every Gatling request. 

We have showed how to create random request in every action using the power of Scala higher order functions. And the most important, we have had so much fun while doing it. 

In the next two posts about Gatling we will discuss about:
* How to deploy this performance test suite into a ECS cluster.
* Verify that the request has been processed by Kafka. Usage of Akka Streams and Akka actors to asyncronously check the response has been successfully processed.