---
title: Scale up and update a Kafka cluster in AWS
layout: post
featured: images/vpc_kafka/kafka_logo.png
description: Detailed steps about how to update and scale up a production Kafka cluster
  without losing any data. How to rebalance your Kafka cluster and update the broker
  instances.
---

One of the most common problems in a Kafka production cluster is how to make updates in the cluster without stopping the cluster, so we ensure zero data loss. 

Kafka by default ensures zero data loss when the replication factor per topic is greater than 1. 

In our case the Kafka cluster has been deployed in AWS using CloudFormation and CodeDeploy. CloudFormation is in charge of creating all the infrastructure while CodeDeploy installs Kafka in our instances. The references to CloudFormation and CodeDeploy are minor during the reading of this post. You can apply the same or similar solutions to your cluster deployed in Azure or to your cluster deployed in your on premises machines.

Why does Kafka needs to be updated? The changes could be introduced as software or hardware basis. 
* Software upgrades
    * Upgrading the Kafka version
    * Modifying some Kafka properties
    * Include security patches...
* Hardware upgrades: these changes are more difficult to tackle:
    * Increase EBS volume size.
    * Scale up the Kafka cluster.
    * Change the EC2 instance type. 

## Software upgrade
A software upgrade normally would just involve doing a *codedeploy:deploy.* Code deploy should be executed with the *DeploymentConfigName* equals to *OneAtTime* and the replication factor of all the topics greater than one.

During some minutes one of our Kafka brokers is down to install the software upgrades. What does it happen to the Kafka cluster? If the offline broker was a leader for some partitions, it is elected a new leader for those partitions. If in other case is not a leader it needs to be in sync with the leader when the broker is up again. 

In case that our Kafka broker is going to be down for a long time, then we may need to do a reassign of the partitions to move them to the online brokers. Normally with software upgrades, reassigning partitions is not required.

After the software upgrade is done, the broker passes from offline to online. It is recommended at this time, to add some delay, like ten minutes. The reason has been explained before: wait until the broker gets in sync with all of its replica partitions. 

So, doing sofware upgrades is relatively easy. The cluster is stress out during the period that the brokers are offline-online due to the broker synchronization, but it is feasible. If the software upgrade involves a kafka version upgrade, I am not sure this procedure would work, as it would involve having some brokers with one version of Kafka and anothers with the upgraded versions. That would need to be tested.

## Hardware upgrade
Doing a hardware upgrade involves tearing down a broker machine and create a new one. If one node is down, the data in Kafka is not lost, because it exists a replication factor. 

The safest way to upgrade the hardware of your kafka cluster is to make the upgrade in chunks of nodes. Lets assume that we have a kafka cluster with n brokers and we are going to use chunks of m brokers to do the upgrade.
1. Reassign the topic partitions to the non-upgraded nodes. Reassign partitions to brokers from  m to n
2. Do the hardware upgrade from broker 0 to broker m-1.
3. Reassign the topic partitions to brokers from 0 to  m-1.
4. Do the hardware upgrade from broker m to n.
5. Reassign the topic partitions to brokers from 0 to  n (total size of the cluster).

It is a bit tedious, but this is the safest. In our case we used the opportunity of scaling up the cluster to do the hardware update. The steps followed were similar, but in our case:
1. The hardware update was done initially to the new scaled brokers.
2. Adding nodes to the cluster has no effect until the partitions are reassigned.
3. Reassign the partitions to the new scaled instances.
4. Hardware upgrade in the initial brokers.
5. Rebalance the partitions to all the brokers.

## Reassigning partitions
Reassigning partitions is a manual process. There is no command in Kafka to reassign all the topics to specific brokers. It has to be done with two Kafka utilities that you can find in the ${Kafka_Home}/bin directory:

```
./bin/kafka-reassign-partitions.sh --zookeeper $zookeeper --topics-to-move-json-file topics_reassign.json --generate --broker-list "5, 6, 7" 
```

This instruction generates the json file that you can use to execute the partitions reassignment. The json file you need to include as a parameter looks like this:

```
cat topics_reassign.json
{"topics":
     [{"topic": "foo1"},{"topic": "foo2"}],
     "version":1
}
```
As you can see in the previous command, one of the parameters is the *broker-list*. These are the brokers where the partitions are reassigned.

This is the command that perform the reassignment. It has as entry one json file, that is exactly the output of the previous command:
```
 .bin/kafka-reassign-partitions.sh --zookeeper $zookeeper --reassignment-json-file ${topic}_reassign_partitions.json --execute
```
To get all the topics of your cluster,  you just need to execute this instruction:
```
.bin/kafka-topics.sh --zookeeper $zookeeper --list
```

In a production environment where it can contain thousand of topics, I do not feel quite comfortable including the reassignment of all the topics in one instruction. I prefer to make one thousand of single reassignments. The following code automate this process:

```
#!/bin/bash
topicsFilename=$1
brokers=$2
zookeeper=$3
topicsString=$(cat $topicsFilename |tr "\n" " ")
topicsArray=($topicsString)
echo "${topicsArray[*]}"
for topic in "${topicsArray[@]}"
do
   echo "Rebalancing $topic"
   echo "{\"topics\":[{\"topic\": \"$topic\"}],\"version\":1}" > /tmp/${topic}_reassign.json 
   /bin/bash /srv/kafka/bin/kafka-reassign-partitions.sh --zookeeper $zookeeper --topics-to-move-json-file /tmp/${topic}_reassign.json --generate --broker-list $brokers | grep -A1 "Proposed partition reassignment configuration" | grep -v "Proposed partition reassignment configuration" > /tmp/${topic}_reassign_partitions.json 
   echo "Applying reassigment of partitions for $topic"
   cat /tmp/${topic}_reassign_partitions.json 

    /bin/bash /srv/kafka/bin/kafka-reassign-partitions.sh --zookeeper $zookeeper --reassignment-json-file /tmp/${topic}_reassign_partitions.json --execute
done
```
Considerations about the previous code:
* This script has 3 parameters: the filename that contains the list of topics, the reassignment brokers and the zookeeper url.
* Line 8: iterating through the topics. Per topic we are going to call the two commands previously explained.
* Line 12: the output of the *kafka-reassign-partitions* is not quite clean. We need to take the last lines before the line *"Proposed partition reassignment configuration"*. With the hep of grep we can do it. Magic!

You can check that your topic partitions have been reassigned
```
.bin/kafka-topics.sh --zookeeper $zookeeper --list
```
## Conclusion
This post contains a real Kafka production problem and it explained how to solve it in a safe way that ensures zero data loss.