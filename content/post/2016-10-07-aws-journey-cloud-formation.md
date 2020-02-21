---
title: AWS journey — Cloud Formation
date: 2016-10-07T00:00:46+01:00
categories:
  - all
tags:
  - aws
  - cloud-formation
---

As a [small start up](https://www.purposefly.com/), we’re playing with lot of technologies and we
try to choose the best ones. Okay, sometimes not because of money constraints, but we’re trying.
One day, can’t remember when it was, we decided to go with AWS. Not just EC2 instances for Docker
Cloud, but full stack. I mean API Gateway, Lambda, EC2, ECS, DynamoDB, etc. Counted them, AWS
provides 51 services. Some of them are perfect, some of them still needs polishing, but they’re
pretty good generally speaking.

Quite huge, isn’t it? Learning curve is pretty steep and you’re going to make lot of mistakes.
Sometimes fatal ones. We, especially I, made them. I’m going to share some of them, some tips, so,
you can avoid head scratching and long night shifts fixing these mistakes when your customers are asleep.

## CloudFormation

AWS CloudFormation gives developers and systems administrators an easy way to create and manage
a collection of related AWS resources, provisioning and updating them in an orderly and predictable fashion.

Don’t start with CloudFormation if you’re new to the AWS platform. It slows you down, because you have
to learn template system, how to use
[_Fn::Join_ & friends](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html),
[nested stack](https://blogs.aws.amazon.com/application-management/post/Tx1T9JYQOS8AB9I/Use-Nested-Stacks-to-Create-Reusable-Templates-and-Support-Role-Specialization),
[outputs](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html),
[parameters](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html),
… You have to learn it along with other AWS services and it’s not an easy task.

Don’t take me wrong. I like CloudFormation, I do use it and I recommend it. What I’m trying to say is — don’t
try to learn everything at once. One by one. Learn whatever service you choose, then jump into the Cloud
Formation and create templates. Why?

* You do not want to learn how to write template when you have no clue how API Gateway & Lambda
  works for example.
* You do not want to wait for deployment (it’s fast, but not enough sometimes), you want to tweak
  settings, play with it and test it immediately.

Also you can easily shoot yourself:

* Resources deployed via CF templates shouldn’t be manually tweaked at all. You can easily update your
  templates, CF can process these updates and it updates resources automatically. If you touch them
  manually, CF can be puzzled (state differs) and you can end up in limbo.
* Never delete resources manually. Update template and let CF process it. If you do it in reverse order,
  CF can be puzzled. You can fix it by recreating resource manually sometimes. Better learn how to work with
  [DeletionPolicy](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html)
  attribute.
* What’s more important. Never ever delete nested stack manually. This action is not reversible.
  You can recreate nested stack with the same name, but it has different PhysicalID and thus the stack
  is there, but it’s not linked to your parent stack.

![Update Rollback Failed](/images/aws/update-rollback-failed.png)

I deleted nested stack manually (frankly, I don’t know why I did it) and I’m angry with myself.

![Update Rollback Failed](/images/aws/update-rollback-failed-2.png)

Hope AWS support guys can help me (tweak resource PhysicalID) and if not I’ll be forced to delete stack,
resources, … and create everything “from scratch”.

“From scratch” in quotes, because we have templates for 99.9999% of our resources. But there are other
things to fix with this redeployment like Lambda functions configuration (another beast, will dive into
details in another article), Continuous Integration (we do use Travis CI), etc.

## Nested stacks

You can split one stack into several stacks and link them together. You can pass output from one stack
as a parameter value to another one, which is really convenient. This is not possible with standalone
stacks. I’m tempted to recommend one root stack for all your stacks. You never know when you’ll need
output of one stack as an input for another one.

We have several root stacks, several nested stacks, but these root stacks are not linked together with
some kind of top level stack for everything. My hope is recently announced
[Fn::ImportValue](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html)
function. But there’s one limitation which bothers me little bit:

> You can’t modify or remove the output value as long as it’s referenced by another stack.

I didn’t find time to test this function. But if I understand it correctly, I can’t change the output
value with template update, because the value is kinda “copied” to another stack and then the initial
stack is locked. If so, not sure how useful this feature is. Because nested stacks, outputs & parameters,
handle this very well. Whenever I change something in one stack, all other dependent stacks are updated
accordingly.

What I’m going to do? Will test _Fn::ImportValue_ and will use it. Or I’ll refactor our stacks and will
add 3rd level (top for all stacks).

## My Cloud Formation workflow

* I manually create resources, tweak them and make them working together. Especially when I’m trying
  to learn new AWS service and I don’t know how to do it properly yet.
* If they’re in acceptable state, they work, I do not expect huge changes, work on CF templates is started.
* Then I delete manually created resources, put these templates in our repo where Travis CI handles them
  automatically (stack creation, updates, …).
* I no longer care about these resources in the AWS Console and I work with templates only.

Skipping first step when I know what I would like to achieve and **mainly** how to do it.

## Conclusion

Use it, highly recommended, but try to learn other services first. Then learn how to describe them in
Cloud Formation templates.

Think twice:

* Don’t create huge, not easily maintainable stacks. Use nested stacks.
* Is there a tiny chance that you’ll need the output value of one stack as an input value for another stack?
  Use nested stacks. (or _Fn::ImportValue_, not tested yet)
* Don’t tweak resources created via CF manually. Don’t delete them as well.
* Learn how _DeletionPolicy_ attribute works.
* Finally, don’t delete nested stack manually = limbo.

Templates allows me to deploy everything in under 15 minutes without worrying if I did or didn’t make
a mistake. Who likes manual and repetitive work? Me not. So, use them.
