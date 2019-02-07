---
title: Exposing Kafka throw different AWS VPCs
layout: post
description: Detailed architecture and implementation of a multi VPC Kafka ecosystem
  in AWS. The implementation is based on VPC endpoints and a NLB to route the traffic
  to the Kafka brokers.
featured: images/vpc_kafka/kafka_logo.png
---

Apache Kafka is an streaming distributed platform that allows companies and organizations to have a central point where data can be streamed. It is a resilient platform that ensures zero data loss with its system of data replication over the kafka cluster. 

Kafka implements the architecture model Publish-Subscriber. This architecture solves the common problem in organizations about having interconnected services around all the organization, what is commonly known as Spaguetti Architecture. 

Without entering in detail about what Kafka works and how the data is partitioned over all the kafka brokers. this is a simple diagram that shows what Kafka does and what it is the Publish-Subscriber model:

<center><img src="/assets/images/vpc_kafka/kafka_model.jpg"/></center>


The main objective of this post is to show how a Kafka cluster deployed in AWS can be migrated from a single VPC to a multi VPC architecture. 

<center><img src="/assets/images/vpc_kafka/basic_architecture_multi_environment.jpg"/></center>

This means that we add flexibility. The producers can be deployed in different VPCs, different AWS accounts and the Kafka cluster can be deployed in its VPC. 

The communication between AWS accounts is achieved by the VPC endpoints. It is created a VPC endpoint Service in the pipeline side that is routing the data to an NLB connected to the Kafka brokers. The producer side communicates with the VPC Endpoint Service using a VPC Endpoint.


<center><img src="/assets/images/vpc_kafka/multi_vpc_detailed_arch.jpg"/></center>
