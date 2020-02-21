---
title: AWS journey — EC2 Container Service
date: 2016-10-20T11:28:46+01:00
categories:
  - all
tags:
  - aws
  - ec2
---

We were using (and we’re still it using for some services) Docker Cloud. Main problem with DC is that
they can’t handle private subnets. EC2 instance must be in the public subnet and that’s not what we want.
Hooray (irony), we have to switch to the EC2 Container Service. Try to explain this to our business
department. Additional time for infrastructure. Can handle it, but it’s tough sometimes.

## Docker images repository

### Cloud Formation Template

This step is optional, but we’re trying to leverage as many as possible AWS services. You need a repository
for your Docker images. And because I like Cloud Formation, here’s the template:

```json
"AthenaRepository": {
  "Type": "AWS::ECR::Repository",
  "Condition": "DevelopmentEnv",
  "Properties": {
    "RepositoryName" : "athena",
    "RepositoryPolicyText" : {
      "Version": "2008-10-17",
      "Statement": [
        {
          "Sid": "ProductionAccountPullAccess",
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::$ANOTHER-ACCOUNT-ID:root"
          },
          "Action": [
            "ecr:GetDownloadUrlForLayer",
            "ecr:BatchGetImage",
            "ecr:BatchCheckLayerAvailability"
          ]
        }
      ]
    }
  }
}
```

Why _RepositoryPolicyText_? We have several AWS accounts, but we want just one repository.
This policy gives pull access to _$ANOTHER-ACCOUNT-ID_ account.

Why _DevelopmentEnv_ condition? Same reason, just one repo per several AWS accounts. We do use same
templates for all accounts. They’re parametrised.

### Push images

There’s nothing new. Just replace _docker login …_ with
[_$(aws ecr get-login)_](http://docs.aws.amazon.com/cli/latest/reference/ecr/get-login.html).

```sh
$(aws ecr get-login)
docker build -t athena .
docker tag athena $AWS_ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/athena:build-$TRAVIS_BUILD_NUMBER
docker push $AWS_ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/athena:build-$TRAVIS_BUILD_NUMBER</pre>
```

## EC2 instance

### Cluster

First of all, you need cluster. There’s no way how to deploy your services without it. Cloud Formation
template:

```json
"ECSCluster" : {
  "Type" : "AWS::ECS::Cluster",
  "Properties" : {
    "ClusterName" : "Athena"
  }
```

Now we have _Athena_ cluster. But it doesn’t contain EC2 instance(s). How to assign them? Fun is coming.

### Role

We have to set EC2 instance profile, which can contain roles. And one of our roles must allow communication
with EC2 Container Service.

```json
"ECSRolePolicy": {
  "Type": "AWS::IAM::ManagedPolicy",
  "Properties": {
    "PolicyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ecr:BatchCheckLayerAvailability",
            "ecr:BatchGetImage",
            "ecr:DescribeRepositories",
            "ecr:GetAuthorizationToken",
            "ecr:GetDownloadUrlForLayer",
            "ecr:GetRepositoryPolicy",
            "ecr:ListImages",
            "ecs:CreateCluster",
            "ecs:DeregisterContainerInstance",
            "ecs:DiscoverPollEndpoint",
            "ecs:Poll",
            "ecs:RegisterContainerInstance",
            "ecs:StartTask",
            "ecs:StartTelemetrySession",
            "ecs:SubmitContainerStateChange",
            "ecs:SubmitTaskStateChange"
          ],
          "Resource": [
            "*"
          ]
        }
      ]
    }
  }
},
"ECSRole": {
  "Type": "AWS::IAM::Role",
  "Properties": {
    "ManagedPolicyArns": [
      { "Ref": "ECSRolePolicy" }
    ],
    "AssumeRolePolicyDocument": {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "ec2.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
  }
},
"InstanceProfile" : {
  "Type" : "AWS::IAM::InstanceProfile",
  "Properties" : {
    "Path" : "/",
    "Roles" : [ { "Ref": "ECSRole" } ]
  }
}
```

We can assign it in this way:

```json
"Athena": {
  "Type": "AWS::EC2::Instance",
  "Properties": {
    ...
    "IamInstanceProfile" : {"Ref" : "InstanceProfile"},
    ...
```

### ECS configuration

Still not done. EC2 instance must know the cluster name. Here we can leverage _UserData_ property:

```json
"Athena": {
  "Type": "AWS::EC2::Instance",
  "Properties": {
    ...
    "UserData": { "Fn::Base64": { "Fn::Join" : [ "n", [
      "#cloud-config",
      "repo_update: true",
      "repo_upgrade: all",
      "package_upgrade: true",
      "packages:",
      " - aws-cli",
      " - ecs-init",
      "runcmd:",
      { "Fn::Join" : [ "", [
        " - echo ECS_CLUSTER=",
        { "Ref": "ECSCluster" },
        " &gt;&gt; /etc/ecs/ecs.config"
      ]]},
      " - service docker start",
      " - start ecs" ]]}},
      ...
```

And …

![Athena ECS Cluster](/images/aws/athena-ecs-cluster.png)

… victory 😉 Registered Container Instances shows 1.

## Service

### Task definition

I’m not going to cover task definition. You can study it
[here](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_defintions.html). It’s pretty
straightforward. Kind of stack file for Docker Cloud, but in JSON. Very similar.

### Travis CI script

First part is task registration (or update with the same command). New revision is created if task
already exists. It’s a simple sequence 1, 2, 3, …

Sad is that I didn’t find a way how to specify revision number manually. Just to keep it in sync with
$_TRAVIS_BUILD_NUMBER_. Next part is more interesting. We try to get service ARN. If not found, we’re
going to create it otherwise we’re going to update it.

```sh
aws ecs register-task-definition --cli-input-json file://athena-task-definition.json

ARN=$(aws ecs describe-services --cluster Athena --services athena | jq '.services? | .[0]? | .serviceArn?' -r -M)
if ["$ARN" = "null"]; then
    aws ecs create-service \
        --cluster Athena \
        --service-name athena \
        --task-definition athena \
        --deployment-configuration maximumPercent=100,minimumHealthyPercent=0 \
        --desired-count 1
else
    aws ecs update-service \
        --cluster Athena \
        --service $ARN \
        --task-definition athena \
        --deployment-configuration maximumPercent=100,minimumHealthyPercent=0 \
        --desired-count 1
fi<
```

> There’s one issue with this script — if you delete service manually, it’s still accessible via
> AWS CLI (state is inactive). Service ARN is found, but update-service is going to fail;
> create-service must be used in this case. Better add state check to this script as well.

Focus on
[minimumHealthyPercent](http://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DeploymentConfiguration.html).
Value _0_ says — feel free to stop my service during deployment. It’s useful in development environment
where you can have one tiny instance for example. If you do not set it to _0_, deployment can fail:

* one tiny instance in cluster,
* not enough memory on your EC2 instance, because old revision is still running,
* port in use, because of the same reason,
* etc.

Do not use this strategy in production environment. You should have more than one instance, setup
[load balancing](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html),
etc.

That’s it. You have pretty decent idea how EC2 Container Service works. Not all issues were covered, but
should be enough to start with it. Hope that helps. It took me some time to find all these things.
