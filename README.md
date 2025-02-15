# SuperGuard deployment infra with Amazon ECS, AWS CloudFormation, and an Application Load Balancer

This reference architecture provides a set of YAML templates for deploying superguard services to [Amazon EC2 Container Service (Amazon ECS)](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) with [AWS CloudFormation](https://aws.amazon.com/cloudformation/).

## Overview

![infrastructure-overview](images/architecture-overview.png)

The repository consists of a set of nested templates that deploy the following:

- A tiered [VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html) with public and private subnets, spanning an AWS region.
- A highly available ECS cluster deployed across two [Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) in an [Auto Scaling](https://aws.amazon.com/autoscaling/) group and that are AWS SSM enabled.
- A pair of [NAT gateways](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html) (one in each zone) to handle outbound traffic.
- Two interconnecting microservices deployed as [ECS services](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) (website-service and product-service).
- An [Application Load Balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/) to the public subnets to handle inbound traffic.
- ALB path-based routes for each ECS service to route the inbound traffic to the correct service.
- Centralized container logging with [Amazon CloudWatch Logs](http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html).
- A [Lambda Function](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) and [Auto Scaling Lifecycle Hook](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html) to [drain Tasks from your Container Instances](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-draining.html) when an Instance is selected for Termination in your Auto Scaling Group.

## Why use AWS CloudFormation with Amazon ECS?

Using CloudFormation to deploy and manage services with ECS has a number of nice benefits over more traditional methods ([AWS CLI](https://aws.amazon.com/cli), scripting, etc.).

#### Infrastructure-as-Code

A template can be used repeatedly to create identical copies of the same stack (or to use as a foundation to start a new stack). Templates are simple YAML- or JSON-formatted text files that can be placed under your normal source control mechanisms, stored in private or public locations such as Amazon S3, and exchanged via email. With CloudFormation, you can see exactly which AWS resources make up a stack. You retain full control and have the ability to modify any of the AWS resources created as part of a stack.

#### Self-documenting

Fed up with outdated documentation on your infrastructure or environments? Still keep manual documentation of IP ranges, security group rules, etc.?

With CloudFormation, your template becomes your documentation. Want to see exactly what you have deployed? Just look at your template. If you keep it in source control, then you can also look back at exactly which changes were made and by whom.

#### Intelligent updating & rollback

CloudFormation not only handles the initial deployment of your infrastructure and environments, but it can also manage the whole lifecycle, including future updates. During updates, you have fine-grained control and visibility over how changes are applied, using functionality such as [change sets](https://aws.amazon.com/blogs/aws/new-change-sets-for-aws-cloudformation/), [rolling update policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html) and [stack policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html).

## Template details

The templates below are included in this repository and reference architecture:

| Template                                                                   | Description                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [iam-setup.yaml](src/iam-setup.yaml)                                       | This is the setup script template - deploy it to CloudFormation and it will generate the required IAM user with roles needed for all the other deployments.                                                                                                                                                                                                                          |
| [master.yaml](master.yaml)                                                 | This is the master template - deploy it to CloudFormation and it includes all of the others automatically.                                                                                                                                                                                                                                                                           |
| [infrastructure/vpc.yaml](infrastructure/vpc.yaml)                         | This template deploys a VPC with a pair of public and private subnets spread across two Availability Zones. It deploys an [Internet gateway](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html), with a default route on the public subnets. It deploys a pair of NAT gateways (one in each zone), and default routes for them in the private subnets. |
| [infrastructure/security-groups.yaml](infrastructure/security-groups.yaml) | This template contains the [security groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html) required by the entire stack. They are created in a separate nested template, so that they can be referenced by all of the other nested templates.                                                                                                       |
| [infrastructure/load-balancers.yaml](infrastructure/load-balancers.yaml)   | This template deploys an ALB to the public subnets, which exposes the various ECS services. It is created in in a separate nested template, so that it can be referenced by all of the other nested templates and so that the various ECS services can register with it.                                                                                                             |
| [infrastructure/ecs-cluster.yaml](infrastructure/ecs-cluster.yaml)         | This template deploys an ECS cluster to the private subnets using an Auto Scaling group and installs the AWS SSM agent with related policy requirements.                                                                                                                                                                                                                             |
| [infrastructure/lifecyclehook.yaml](infrastructure/lifecyclehook.yaml)     | This template deploys a Lambda Function and Auto Scaling Lifecycle Hook to drain Tasks from your Container Instances when an Instance is selected for Termination in your Auto Scaling Group.                                                                                                                                                                                        |
| [services/product-service/service.yaml](services/*/service.yaml)           | These are the application specific stacks                                                                                                                                                                                                                                                                                                                                            |

After the CloudFormation templates have been deployed, the [stack outputs](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html) contain a link to the load-balanced URLs for each of the deployed service.

The ECS instances should also appear in the Managed Instances section of the EC2 console.

## How do I...?

### Create the required IAM user

1. Deploy the setup template

```bash
$ aws cloudformation deploy \
  --stack-name github-actions-cloudformation-deploy-setup \
  --template-file src/iam-setup.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-southeast-2 \
  --profile sg
```

2. Get the newly created user secrets

```bash
$ aws secretsmanager get-secret-value \
  --secret-id github-actions-cloudformation-deploy \
  --region ap-southeast-2 \
  --query SecretString \
  --output text \
  --profile sg
```

3. Add the returned `AccessKeyId`, and `SecretAccessKey` as Github Secrets in the service repository

### Create a new ECS service

1. Push your container to a registry somewhere (e.g., [Amazon ECR](https://aws.amazon.com/ecr/)).
2. Copy one of the existing service templates in [services/\*](/services).
3. Update the `ContainerName` and `Image` parameters to point to your container image instead of the example container.
4. Increment the `ListenerRule` priority number (no two services can have the same priority number - this is used to order the ALB path based routing rules).
5. Copy one of the existing service definitions in [master.yaml](master.yaml) and point it at your new service template. Specify the HTTP `Path` at which you want the service exposed.
6. Deploy the templates as a new stack, or as an update to an existing stack.

### Setup centralized container logging

By default, the containers in your ECS tasks/services are already configured to send log information to CloudWatch Logs and retain them for 365 days. Within each service's template (in [services/\*](services/)), a LogGroup is created that is named after the CloudFormation stack. All container logs are sent to that CloudWatch Logs log group.

You can view the logs by looking in your [CloudWatch Logs console](https://console.aws.amazon.com/cloudwatch/home?#logs:) (make sure you are in the correct AWS region).

ECS also supports other logging drivers, including `syslog`, `journald`, `splunk`, `gelf`, `json-file`, and `fluentd`. To configure those instead, adjust the service template to use the alternative `LogDriver`. You can also adjust the log retention period from the default 365 days by tweaking the `RetentionInDays` parameter.

For more information, see the [LogConfiguration](http://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_LogConfiguration.html) API operation.

### Change the ECS host instance type

This is specified in the [master.yaml](master.yaml) template.

By default, [t2.large](https://aws.amazon.com/ec2/instance-types/) instances are used, but you can change this by modifying the following section:

```
ECS:
  Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: ...
      Parameters:
        ...
        InstanceType: t2.large
        InstanceCount: 4
        ...
```

### Adjust the Auto Scaling parameters for ECS hosts and services

The Auto Scaling group scaling policy provided by default launches and maintains a cluster of 4 ECS hosts distributed across two Availability Zones (min: 4, max: 4, desired: 4).

It is **_not_** set up to scale automatically based on any policies (CPU, network, time of day, etc.).

If you would like to configure policy or time-based automatic scaling, you can add the [ScalingPolicy](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-as-policy.html) property to the AutoScalingGroup deployed in [infrastructure/ecs-cluster.yaml](infrastructure/ecs-cluster.yaml#L69).

As well as configuring Auto Scaling for the ECS hosts (your pool of compute), you can also configure scaling each individual ECS service. This can be useful if you want to run more instances of each container/task depending on the load or time of day (or a custom CloudWatch metric). To do this, you need to create [AWS::ApplicationAutoScaling::ScalingPolicy](http://docs.aws.amazon.com/pt_br/AWSCloudFormation/latest/UserGuide/aws-resource-applicationautoscaling-scalingpolicy.html) within your service template.

### Deploy multiple environments (e.g., dev, test, pre-production)

Deploy another CloudFormation stack from the same set of templates to create a new environment. The stack name provided when deploying the stack is prefixed to all taggable resources (e.g., EC2 instances, VPCs, etc.) so you can distinguish the different environment resources in the AWS Management Console.

### Change the VPC or subnet IP ranges

This set of templates deploys the following network design:

| Item           | CIDR Range     | Usable IPs | Description                                        |
| -------------- | -------------- | ---------- | -------------------------------------------------- |
| VPC            | 10.180.0.0/16  | 65,536     | The whole range used for the VPC and all subnets   |
| Public Subnet  | 10.180.8.0/21  | 2,041      | The public subnet in the first Availability Zone   |
| Public Subnet  | 10.180.16.0/21 | 2,041      | The public subnet in the second Availability Zone  |
| Private Subnet | 10.180.24.0/21 | 2,041      | The private subnet in the first Availability Zone  |
| Private Subnet | 10.180.32.0/21 | 2,041      | The private subnet in the second Availability Zone |

You can adjust the CIDR ranges used in this section of the [master.yaml](master.yaml) template:

```
VPC:
  Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub ${TemplateLocation}/infrastructure/vpc.yaml
      Parameters:
        EnvironmentName:    !Ref AWS::StackName
        VpcCIDR:            10.180.0.0/16
        PublicSubnet1CIDR:  10.180.8.0/21
        PublicSubnet2CIDR:  10.180.16.0/21
        PrivateSubnet1CIDR: 10.180.24.0/21
        PrivateSubnet2CIDR: 10.180.32.0/21
```

### Update an ECS service to a new Docker image version

ECS has the ability to perform rolling upgrades to your ECS services to minimize downtime during deployments. For more information, see [Updating a Service](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html).

To update one of your services to a new version, adjust the `Image` parameter in the service template (in [services/\*](services/) to point to the new version of your container image. For example, if `1.0.0` was currently deployed and you wanted to update to `1.1.0`, you could update it as follows:

```
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    ContainerDefinitions:
      - Name: your-container
        Image: registry.example.com/your-container:1.1.0
```

After you've updated the template, update the deployed CloudFormation stack; CloudFormation and ECS handle the rest.

To adjust the rollout parameters (min/max number of tasks/containers to keep in service at any time), you need to configure `DeploymentConfiguration` for the ECS service.

For example:

```
Service:
  Type: AWS::ECS::Service
    Properties:
      ...
      DesiredCount: 4
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 50
```

### Use the SSM Run Command function to see details in the ECS instances

The AWS SSM Run Command function, in the EC2 console, can be used to execute commands at the shell on the ECS instances. These can be helpful for examining the installed configuration of the instances without requiring direct access to them.

### Spot Instances and the Hibernate Agent.

In order to use Spot with this template, you will need to enable `SpotPrice` under the `AWS::AutoScaling::LaunchConfiguration` or add in `AWS::EC2::SpotFleet` support. To fully use Hibernation with Spot instances, please review [Spot Instance Interruptions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html).

## License

Copyright 2011-2016 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

[http://aws.amazon.com/apache2.0/](http://aws.amazon.com/apache2.0/)

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

TODO:

- support services with different instance type requirments
