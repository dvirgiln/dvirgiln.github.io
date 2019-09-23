---
title: Migration from EC2  to Fargate
featured: images/aws_logo.png
description: This post is a good introduction to AWS ECS and more particularity to the Fargate deployment type. We will explain how Fargate works, differences with AWS ECS EC2 and how to migrate from ECS EC2
  services to ECS Fargate.
layout: post
---

ECS stands for *Elastic Container Service* and its function allows to run docker applications in a AWS cluster. The docker applications hav to be deployed in AWS Elastic Container Registry(ECR).

The first thing to do when creating/running a docker application in ECS is to create an ECS cluster.

<center><img src="/assets/images/ecs/create_cluster.png"/></center>

There are two cluster flavours:

* **EC2**: when you create an EC2 Linux cluster or EC2 Windows cluster, it allows as well to deploy the services in Fargate (Networking in the previous image) mode. During the Create Cluster wizard you have the option to create new VPC and Subnets or select existing ones. As well you need to define your EC2 instance type and number of instances.
* **Fargate**: this is the serverless approach. You do not need to provision any resource. The services deployed in Fargate could scale out  without almost any restriction (we will cover some restrictions of Fargate in a few seconds).

If you have your services already deployed in EC2 and you want to deploy them in Fargate, for example to avoid having an overprovisioned cluster, or to avoid problems related EBS volume space, there are a few steps to have in consideration.

As you know, when you try your CF template, there is not a way to know if that is correct. You have to include the change and try it. AWS is not providing you a full set of errors in your template. The CF output just shows the first error encountered. That's why this guide is useful if your goal is to migrate from EC2 cluster to Fargate. Steps to follow:

This is how our CloudFormation ECS task definition looks like before migrating to Fargate:
```
 ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !Ref RoleArn
      Volumes:
        -
          Name: TMP_VOLUME
      ContainerDefinitions:
        -
          Name: !Ref ContainerName
          Essential: true
          Image: !Ref DockerImage
          Cpu: !Ref ECSCpu
          Memory: !Ref ECSMemory
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref CloudWatchLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: task
          MountPoints:
            -
              ContainerPath: /srv/kafka-connect/temp
              SourceVolume: TMP_VOLUME
              ReadOnly: false
          PortMappings:
            -
              ContainerPort: !Ref ContainerPort
              HostPort: 0 # ECS will dynamically assign a port from the ephemeral range.
              Protocol: !Ref ContainerProtocol
      Family: !Ref Function

  ECSService:
    Type: AWS::ECS::Service
    DependsOn:
      - Listener7400
    Properties:
      Cluster: !Ref Cluster
      DeploymentConfiguration:
        MinimumHealthyPercent: 50
      DesiredCount: !Ref DesiredCount
      HealthCheckGracePeriodSeconds: 30
      LoadBalancers:
        -
          ContainerName: !Ref ContainerName
          ContainerPort: !Ref ContainerPort
          TargetGroupArn: !Ref TargetGroupHTTP
      Role: !Sub
        - arn:aws:iam::${AWS::AccountId}:${role}
        -
          role: !FindInMap
            - EnvironmentMap
            - !Ref Environment
            - ecsServiceSchedulerRole
      TaskDefinition: !Ref ContainerTask
      ServiceName: !Ref ServiceName
```
First change to apply is the **Launch Type** in the **ECSService** component. This change is the first and more intuitive:
```
  ECSService:
    Type: AWS::ECS::Service
    Properties:
      LaunchType: FARGATE
```
The first error that we encountered was *Fargate requires task definition to have execution role ARN to support log driver awslogs.*. This error is fixed by adding an execution role as part of the  **AWS::ECS::TaskDefinition** :
```
  ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !Ref RoleArn
      ExecutionRoleArn: !Ref RoleArn
```
Before it was only defined a TaskRoleArn.

The next error that we found was *When networkMode=awsvpc, the host ports and container ports in port mappings must match.*. To solve this problem you need to include a change in the **AWS::ECS::TaskDefinition**:
```
          PortMappings:
            -
              ContainerPort: !Ref ContainerPort
              #HostPort: 0
              Protocol: !Ref ContainerProtocol
```
As you see the line 4 has been commented. In Fargate the *host port *and *container port* should have the same value. When it is used EC2, as there are many task of the same service deployed in the same ec2 instance the real target port is taken from the ephemeral ports range. If you do not define the hostPort property Fargate will use the same that was defined for the *ContainerPort*.

The next error is *Fargate only supports network mode ?awsvpc?*.

And this is how it looks after having included all the changes already mentioned:
```
  ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
			NetworkMode: awsvpc
```
The *network mode* property. The default value is **bridge**, and this is suitable for ECS EC2 task definitions, but in case of Fargate Bridge is not supported, and the only option is **awsvpc**. The bridge network mode allows using the Docker bridge to communicate docker containers that are running in the same EC2 instance, while in the AWS VPC network mode, every task has its own ENI, this means, every task has its own ipv4 from the subnet that it has been deployed.

This is one of the restrictions that we can see between EC2 and Fargate. In Fargate every task has its own IPV4. In case of running a long number of tasks, it could happen that subnets can run out of ips.

After including the last change, the next error found was *Task definition does not support launch_type FARGATE.*. To solve this problem it is just required to add another field in the task definition:
```
  ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
		      RequiresCompatibilities:
						- FARGATE
```
By default the ECS tasks are only compatible with EC2 deployments.

We are in the middle of the migration to Fargate. After our last change, the output is *Fargate requires that 'cpu' be defined at the task level.* . So, even though it is defined the CPU as part of the task definition, for Fargate we need to declare it as part of the service:
```
  ECSService:
    Type: AWS::ECS::Service
    Properties:
			Cpu: !Ref ECSCpu
```
The next error found was *The provided target group arn:aws:elasticloadbalancing:us-west-2:${ACCOUNT}:targetgroup/kconn-Targe-sssss/ has target type instance, which is incompatible with the awsvpc network mode specified in the task definition.*.

We do need to redefine the Target Group when deploying to Fargate:
```
  TargetGroupHTTP:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      Matcher:
        HttpCode: 200
      #Port: 1 # ECS will dynamically assign ports and override this dummy value.
      Port: 7400
      TargetType: ip
```
It is required to define the *Port* and the *TargetType: ip*. When the deployment is done with EC2, the port is set to 1, and this tells ECS that it has to dynamically assign the port(the ephemeral port).

Once this error is fixed, the next error is *Network Configuration must be provided when networkMode 'awsvpc' is specified.*
```
  ECSService:
    Type: AWS::ECS::Service
    Properties:
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref ServerSubnets
          SecurityGroups:
            - !Ref FargateSecurityGroup
  FargateSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - CidrIp: !Ref VPCCidr
          FromPort: 7400
          IpProtocol: tcp
          ToPort: 7400       
```

It requires to add a *Security Group* with the port defined in the **PortMappings** property to allow access to the service, in our case from within the VPC.

When the Network Configuration is defined in the *ECS Service* component, then it is required to remove the Role property. Talk here about the ECS SERVICE role  {TO_BE_DONE}

And finally, this is the last error that we found: *Status reason	CannotPullECRContainerError: AccessDeniedException: User: arn:aws:sts::${ACCOUNT}:assumed-role/name/... is not authorized to perform: ecr:GetAuthorizationToken on resource*.

To solve this problem as part of the role associated to the task, you need to add the following policy:
```
-
  PolicyName: 'ecr-access'
  PolicyDocument:
    Statement:
      -
        Action:
          - 'ecr:GetAuthorizationToken'
          - 'ecr:BatchCheckLayerAvailability'
          - 'ecr:GetDownloadUrlForLayer'
          - 'ecr:BatchGetImage'
        Effect: Allow
        Resource: '*'
```

And that's all... This is how our CloudFormation template looks like after all the changes:
```
  ContainerTask:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !Ref RoleArn
      ExecutionRoleArn: !Ref RoleArn
      Cpu: !Ref ECSCpu
      Memory: !Ref ECSMemoryHardLimit
      RequiresCompatibilities:
        - FARGATE
      NetworkMode: awsvpc
      Volumes:
        -
          Name: TMP_VOLUME
      ContainerDefinitions:
        -
          Name:  !Ref ContainerName
          Essential: true
          Image: !Ref DockerImage
          Cpu: !Ref ECSCpu
          Memory: !Ref ECSMemoryHardLimit
          MemoryReservation: !Ref ECSMemorySoftLimit
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref CloudWatchLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: task
          MountPoints:
            -
              ContainerPath: /srv/kafka-connect/temp
              SourceVolume: TMP_VOLUME
              ReadOnly: false
          PortMappings:
            -
              ContainerPort: !Ref ContainerPort
              #HostPort: 0 # ECS will dynamically assign a port from the ephemeral range.
              Protocol: !Ref ContainerProtocol
      Family: !Ref Function

  ECSService:
    Type: AWS::ECS::Service
    DependsOn:
      - Listener7400
      - FargateSecurityGroup
    Properties:
      LaunchType: FARGATE
      Cluster: !Ref Cluster
      DeploymentConfiguration:
        MinimumHealthyPercent: 50
      DesiredCount: !Ref DesiredCount
      HealthCheckGracePeriodSeconds: 1200
      LoadBalancers:
        -
          ContainerName: !Ref ContainerName
          ContainerPort: !Ref ContainerPort
          TargetGroupArn: !Ref TargetGroupHTTP
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref ServerSubnets
          SecurityGroups:
            - !Ref FargateSecurityGroup
      TaskDefinition: !Ref ContainerTask
      ServiceName: !Ref ServiceName

```
### Conclusions

As you notice, changing from EC2 to Fargate is not trivial. The documentation is not clear about it. In my opinion it would be easier if, instead of having one CloudFormation component that works for both Fargate and EC2 clusters, it could have been decoupled into *ECS::FargateService* and *ECS::EC2Service* and the same would apply for the tasks. With this solution we would avoid having if else conditions in the documentation for almost all the attributes.

In case you have to migrate your services to Fargate, I hope this guide will help you.
