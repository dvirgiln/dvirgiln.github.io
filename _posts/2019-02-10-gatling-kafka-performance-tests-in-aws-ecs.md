---
title: Gatling Kafka performance tests in AWS ECS
description: How to scale out the Kafka performance tests using AWS ECS tasks. Push
  your Gatling performance tests to a new level.
layout: post
featured: images/Gatling-dark-logo.png
---

Gatling is a performance scala library that facilitates running performance tests on your web services/applications. By default Gatling is oriented to HTTP Rest requests. In a previous post we talked about [how to create a custom Gatling performance tests](gatling-custom-kafka-performance-tests) library for your application.

The problem I faced when I wanted to stress Kafka was the number of requests that Gatling can handle per second. Gatling runs on top of a single/multi core CPU instance and even if you increase the number of requests, for instance to 500 thousands per seconds, Gatling is not going to be able to process those requests. At the end every request runs in a single CPU clock time slot, and to increase the parallelism the only way is to deploy Gatling in a machine with more CPU cores.

So deploying the Gatling performance tests into a fat instance is a solution, but in my experience to stress out Kafka with more than 200 thousands MBs/second we need quite a high level of parallelism, more than what can be provided by a single instance. What could we do? **ECS** at the rescue.

We could scale out the performance tests. So I decided that I could just **dockerize** the tests and run them as ECS tasks. Using a ECS cluster, we could scale out our performance tests as we desire.

 ## Dockerizing our app
 Dockerizing a scala app is quite straightforward. They are just 2 files required, DockerFile and an entrypoint.sh.
 
 This is the Docker file:
 
 ```
 FROM centos:7

# Set environment variables
ENV \
    # Java build versions
    JDK_MAJOR="8" \
    JDK_MINOR="131" \
    JDK_BUILD="11" \
    JDK_URL_HASH="d54c1d3a095b4ff2b6607d096fa80163" \
		BASE_SCENARIO=TopicRequest \
    ACTION_SCENARIO=WriteKafka \
    GATLING_NUMBER_USERS_PER_SECOND=1000 \
    GATLING_MAX_DURATION_SECONDS=150 \
    GATLING_DURATION_SECONDS=100 \
    GATLING_NUMBER_RECORDS_PER_TRANSACTION=20


ENV JAVA_HOME="/opt/java/jdk1.${JDK_MAJOR}.0_${JDK_MINOR}"
ENV SCALA_VERSION 2.11.8
ENV SBT_VERSION 0.13.17
ENV SCALA_INST /usr/local/share
ENV SCALA_HOME $SCALA_INST/scala

ADD src ${BASE_FOLDER}/src
ADD project ${BASE_FOLDER}/project
ADD build.sbt ${BASE_FOLDER}
COPY script/entrypoint.sh /entrypoint.sh
RUN chmod +x  /entrypoint.sh


RUN curl -v -L -O -H "Cookie: oraclelicense=accept-securebackup-cookie" \
        http://download.oracle.com/otn-pub/java/jdk/${JDK_MAJOR}u${JDK_MINOR}-b${JDK_BUILD}/${JDK_URL_HASH}/jdk-${JDK_MAJOR}u${JDK_MINOR}-linux-x64.tar.gz
RUN mkdir /opt/java \
    && tar -zxvf jdk-${JDK_MAJOR}u${JDK_MINOR}-linux-x64.tar.gz -C /opt/java \
    && rm -rf \
        ${JAVA_HOME}/javafx-src.zip \
        ${JAVA_HOME}/src.zip \
        jdk-${JDK_MAJOR}u${JDK_MINOR}-linux-x64.tar.gz
RUN yum install -y  python2 python-pip
RUN pip install awscli
RUN \
  curl -fsL http://downloads.typesafe.com/scala/$SCALA_VERSION/scala-$SCALA_VERSION.tgz | tar xfz - -C $SCALA_INST && \
  ln -sf scala-$SCALA_VERSION $SCALA_HOME && \
  echo 'export PATH=$SCALA_HOME/bin:$PATH' > /etc/profile.d/scala.sh
RUN \
  curl https://bintray.com/sbt/rpm/rpm | tee /etc/yum.repos.d/bintray-sbt-rpm.repo

ENV PATH $PATH:$JAVA_HOME/bin


# install sbt
RUN yum install wget -y
RUN wget -O /usr/local/bin/sbt-launch.jar http://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt-launch/$SBT_VERSION/sbt-launch.jar
ADD script/sbt.sh /usr/local/bin/sbt
RUN chmod 755 /usr/local/bin/sbt
RUN sbt sbtVersion
ENTRYPOINT ["/entrypoint.sh"]
```

Considerations about the Dockerfile:
* It installs java, scala and sbt.
* In the lines 10 and 11 are defined the ActionScenario and the Spec to be executed. This is being used in the entrypoint to specify the spec we want to run with the Docker run. Parametrizing everything help us to dinamycally run whatever we want without rebuilding the docker image, just overriding the default environment variables.
* It install as well the awscli. The reason is because we want to **publish the logs is S3**.
* In the line 57, it is the instruction to the entrypoint. The entrypoint is the action or sequence of commands that are executed when the docker image is run (*docker run*)

Let's move on to the entrypoint:

```
#!/usr/bin/env bash
echo "Beginning EntryPoint"
sed -i "s/{{GATLING_DURATION_SECONDS}}/${GATLING_DURATION_SECONDS}/" ${BASE_FOLDER}/config/application.conf
sed -i "s/{{GATLING_MAX_DURATION_SECONDS}}/${GATLING_MAX_DURATION_SECONDS}/" ${BASE_FOLDER}/config/application.conf
sed -i "s/{{GATLING_NUMBER_USERS_PER_SECOND}}/${GATLING_NUMBER_USERS_PER_SECOND}/" ${BASE_FOLDER}/config/application.conf
sed -i "s/{{GATLING_NUMBER_RECORDS_PER_TRANSACTION}}/${GATLING_NUMBER_RECORDS_PER_TRANSACTION}/" ${BASE_FOLDER}/config/application.conf
export KAFKA_HEAP_OPTS="-Xms1G -Xmx2G"
cd ${BASE_FOLDER}
sbt compile "gatling-it:testOnly com.ean.dcp.avro.gatling.${ACTION_SCENARIO}${BASE_SCENARIO}Spec"
echo "Copying simulation results to S3 bucket avrokafkaperfresults_bucket ..."
aws s3 cp target/gatling-it/* s3://avrokafkaperfresults_bucket --recursive
echo "End EntryPoint"
```

Considerations about the entrypoint.sh
* From line 3 to 6, we are just upgrading the default values for our spec. Parametrizing the four parameters in docker allows to dinamycally run different test setups from the AWS ECS Task easily.
* In the line 9, the sbt gatling:testOnly is executing just one specific spec. This allows to test from ECS the Gatling performance test that we desire, just changing the Docker environment variable for *ACTION_SCENARIO* or *BASE_SCENARIO*.
* In the line 11 the gatling performance tests resuts are being uploaded to an s3 bucket.

To build and run locally the performance tests use this:

```
docker build -t kafka_avro_perf_test
docker run kafka_avro_perf_test
```

Once we verify that everything works fine locally we are ready to go and deploy it in ECS. Let's take a look to the *CloudFormation* templates required to run our performance tests as ECS tasks.

 ## ECS CloudFormation templates
 To deploy a Docker image into AWS we need to first have an **ECS ECR** repository. 
 
 ```
   Repository:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: !Ref Function
      RepositoryPolicyText:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              AWS:
                - "arn:aws:iam::${YOUR_AWS_ACCOUNT_NUMBER}:root" 
            Action:
              - "ecr:*"
 ```
 
 Once the repository is created, you would be able to push your image to the ECR repository:
 
 ```
 docker build -t kafka_avro_perf_test
 docker push aws_account_id.dkr.ecr.region.amazonaws.com/kafka_avro_perf_test
 ```
 
 The next step is to create the Iam Roles required to run the ECS task. And IAM role is composed by IAM policies. In our case we need to create a policy to access the S3 bucket where the performance tests results are saved.:
 
 ```
   ECSRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
          -
            Action:
              - sts:AssumeRole
            Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
      RoleName: !Sub ${Function}
      ManagedPolicyArns:
        -
          !ImportValue LogGroupsDenyCreationPolicy

  S3Policy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyDocument:
        Statement:
          -
            Action:
              - 's3:ListBucket'
              - 's3:GetObject'
              - 's3:GetObjectAcl'
              - 's3:PutObject'
              - 's3:PutObjectAcl'
            Effect: Allow
            Resource:
              - !Sub arn:aws:s3:::${StatsBucket}
              - !Sub arn:aws:s3:::${StatsBucket}/*
      PolicyName: s3-access
      Roles:
        - !Ref ECSRole
 ````
 
 Considerations about the previous CF code:
* Line 2: it is created a IAM role *ECSRole*. Then we add policies to that IAM role.
* Line 17: it is created a S3 bucket policy to let the ECS task to write into the stats bucket that we suppose that is already created in S3.
 
 
 Once it is defined the required IAM role and the Docker image is deployed into ECR, we just need to deploy with CF the ECS task. This is the code.
 ```
   CloudWatchLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub ${Function}
      RetentionInDays: !Ref CloudWatchRetentionDays
 
   ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !Ref ECSRole
      RequiresCompatibilities:
        - "EC2"
      ContainerDefinitions:
        -
          Essential: true
          Image: !Ref DockerImage
          MemoryReservation: !Ref Memory
          Name: !Sub ${Function}
          Cpu: !Ref Cpu
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref CloudWatchLogGroup
              awslogs-region: !Ref AWS::Region
      Family: !Sub ${Function}
 ```
 
The previous code has some details to go into:
* In the line 10 it is injected the previously defined ECS IAM role.
* In the line 16 it is injected the docker image already pushed to ECR. It should have the same name as it was pushed into ECR.
* In the line 11 it is specified the task compatibility to be EC2. There are 2 kind of deployments in ECS. You can either deploy to a Fargate or EC2 cluster. In case you decide to deploy the ECS task into a Fargate cluster you need to include a parameter *NetworkMode* with the value *awsvpc*. For more information you can visit [AWS ECS Task CF](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html#cfn-ecs-taskdefinition-networkmode) .

About why using a EC2 ECS cluster or a Fargate one goes far away from the goal of this post. We may have to revisit this topic in future posts.

Once the previous CF templates are being deployed we just need to run our ECS task. Here we have 2 options:
* Manual run: using the AWS ECS console we can easily run the ECS task selecting one ECS EC2 cluster.
* Run from the *AWS CLI*: it facilitates the automation and for instance it can be integrated in a CI platform. 

Obviously we recommend the second option. This is the code to run an ECS task from the AWS CLI:
```
export OVERRIDES_DEFAULT=$(echo "{ \"containerOverrides\": [ { \"name\": \"kafka-avro-performance\", \"environment\": [ { \"name\": \"ACTION_SCENARIO\", \"value\": \"$GATLING_SCENARIO_ACTION\" }, { \"name\": \"GATLING_DURATION_SECONDS\", \"value\": \"$GATLING_DURATION_SECONDS\" },{ \"name\": \"GATLING_NUMBER_RECORDS_PER_TRANSACTION\", \"value\": \"$GATLING_NUMBER_RECORDS_PER_TRANSACTION\" }, { \"name\": \"GATLING_MAX_DURATION_SECONDS\", \"value\": \"$GATLING_MAX_DURATION_SECONDS\" }, { \"name\": \"GATLING_NUMBER_USERS_PER_SECOND\", \"value\": \"$GATLING_NUMBER_USERS_PER_SECOND\" } ] } ]}")

aws ecs run-task --cluster ECSEC2Cluster --launch-type EC2  --overrides "$OVERRIDES_DEFAULT" --count ${NUMBER_OF_TASKS} --task-definition kafka-avro-perf
```
In the previous code we can see 2 parts:
* First it is defined the json object that overrides the default values of our Docker image.
* The last line contains the aws cli instruction to run an ecs task. It contains 4 parameters:
    * Cluster to be deployed
    * Task to deploy
    * Number of task to run
    * Json object that contains the environment variables to override from the docker image.

This last code has been run from Atlassian Bamboo. That's why you can see all the values containing the $ symbol. The reason is because they has been defined as Bamboo variables, to provide more automation and less effort while running the performance tests.

## Conclusions
Dockerizing and deploying an application into ECS could seem to be a bit tedious and complicated. But once you dockerized once, you always tend to dockerize everything. The reason is because everything that is running in a Docker container, you are 100% sure it is going to work no matter which environment it is working on. Docker creates their own environment and download and install all the required dependencies. 

In our case, dockerizing the gatling performance tests was not very difficult. It was just tedious as it required: creating a Docker file, then creating the ECR repository, then the IAM roles, then the ECS task CF code. And then it required to run the ECS task, either manually or from an Continuous Integration plan.

In the next Gatling performance post, we will explore how to create asyncronous and responsive Gatling test using Akka Actors and Akka Streaming.