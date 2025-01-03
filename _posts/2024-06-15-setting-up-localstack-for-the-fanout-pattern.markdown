---
layout: post
title: "Setting up LocalStack for the Fanout Pattern based on AWS SQS and SNS"
date: 2024-06-15 10:00:00 +0100
categories: jekyll update
---

# Introduction
One of the most frequent integrations that I set up locally during my career was the integration 
of Java services with AWS SQS and SNS. Usually, it was needed for testing some synthetic scenarios 
that would have been hard to simulate in test environments.

Some time ago I found out about LocalStack and its capabilities. 
While there are different guides on how to set it up and how to set up SQS and SNS integration with it, 
from my perspective, most of them are a bit more complex than it’s needed, so I decided to write this small simple, 
and complete guide that will show how to configure integration with SQS and SNS using LocalStack Community edition. 
As an example, we will take the [Fanout Pattern](https://en.wikipedia.org/wiki/Fan-out_(software)) 
because it can be simply implemented using the mentioned tools and suits the goal well.

# Setting up LocalStack
Across all the installation methods I found the one using docker-compose the simplest and enough configurable, 
so I will use it:

```yaml
services:
  localstack:
    container_name: localstack
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    environment:
      - SERVICES=sqs,sns
```

This configuration contains only necessary things. It can be adjusted further according to the needs, but basically, 
we can start with this one. Now, we just need to invoke `docker-compose up` in the directory with the created file 
to start LocalStack.

# Configuring AWS profile
To be able to invoke AWS commands in LocalStack, we need to have a configured AWS profile. 
I would recommend having a dedicated profile for testing with LocalStack. Let’s set up one:

```shell
$ aws configure --profile localstack

AWS Access Key ID [None]: test
AWS Secret Access Key [None]: test
Default region name [None]: us-east-1
Default output format [None]: json
```

# Creating queues, topics, and subscriptions
Now, it’s time to check that LocalStack and the AWS profile are configured properly. 
For that, we will create some queues and topics. Then we will subscribe the queues to the topics. 
This will be our simple fanout pattern implementation.

Let’s start with the queues. In the scripts below besides queue names, we also specify an endpoint URL that should 
point to the LocalStack address. Along with that, we specify to use our profile dedicated to LocalStack.

```shell
$ aws sqs create-queue --queue-name queue-1 --endpoint-url=http://localhost:4566 --profile localstack
{
    "QueueUrl": "http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-1"
}

$ aws sqs create-queue --queue-name queue-2 --endpoint-url=http://localhost:4566 --profile localstackaws sqs create-queue --queue-name queue-2 --endpoint-url=http://localhost:4566 --profile localstack
{
    "QueueUrl": "http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-2"
}
```

Next, we will create a topic to distribute messages to the queues:

```shell
$ aws sns create-topic --name topic-1 --endpoint-url=http://localhost:4566 --profile localstack
{
    "TopicArn": "arn:aws:sns:us-east-1:000000000000:topic-1"
}
```

The last step is to subscribe the queues to the topic, but firstly we need to find out their ARNs. 
We will use the queue URLs provided to us during queues creation.
```shell
$ aws sqs get-queue-attributes --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-1 --attribute-names QueueArn --endpoint-url=http://localhost:4566 --profile localstack
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:000000000000:queue-1"
    }
}

$ aws sqs get-queue-attributes --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-2 --attribute-names QueueArn --endpoint-url=http://localhost:4566 --profile localstack
{
    "Attributes": {
        "QueueArn": "arn:aws:sqs:us-east-1:000000000000:queue-2"
    }
}
```

Now, we are ready to subscribe the queues to the topic. I will use `RawMessageDelivery=true` for simplicity:
```shell
$ aws sns subscribe --topic-arn arn:aws:sns:us-east-1:000000000000:topic-1 --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:queue-1 --attributes RawMessageDelivery=true --endpoint-url=http://localhost:4566 --profile localstack
{
    "SubscriptionArn": "arn:aws:sns:us-east-1:000000000000:topic-1:861c7e8a-f85f-48a5-962a-01a78b777692"
}

$ aws sns subscribe --topic-arn arn:aws:sns:us-east-1:000000000000:topic-1 --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:000000000000:queue-2 --attributes RawMessageDelivery=true --endpoint-url=http://localhost:4566 --profile localstack
{
    "SubscriptionArn": "arn:aws:sns:us-east-1:000000000000:topic-1:c179732d-c170-471a-a30a-b3d6dd6b1b78"
}
```

# Testing
We will again use AWS CLI to test our implementation. The scenario is simple: publish a message to the topic 
and make sure that it can be read from both of the queues.

Let’s start with publishing:
```shell
$ aws sns publish --topic-arn "arn:aws:sns:us-east-1:000000000000:topic-1" --message "Hello, World!" --endpoint-url=http://localhost:4566 --profile localstack
{
    "MessageId": "a250e010-588d-4a69-bc0e-4469f0a458af"
}
```
Now, we should be able to retrieve the message from the queues:
```shell
$ aws sqs receive-message --queue-url=http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-1 --endpoint-url=http://localhost:4566 --profile localstack
{
    "Messages": [
        {
            "MessageId": "e3d85f8e-dc35-4ad4-ab3c-7aff82dc9af9",
            "ReceiptHandle": "ZWEyNjBkZGUtYmI3NC00YjE5LWI2ODItYWY5NDUzYThmZmQyIGFybjphd3M6c3FzOnVzLWVhc3QtMTowMDAwMDAwMDAwMDA6cXVldWUtMSBlM2Q4NWY4ZS1kYzM1LTRhZDQtYWIzYy03YWZmODJkYzlhZjkgMTcxMDE0NTk3MC45NzA1NDk=",
            "MD5OfBody": "65a8e27d8879283831b664bd8b7f0ad4",
            "Body": "Hello, World!"
        }
    ]
}

$ aws sqs receive-message --queue-url=http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/queue-2 --endpoint-url=http://localhost:4566 --profile localstack
{
    "Messages": [
        {
            "MessageId": "d417d16f-b627-434f-9994-a297d6c6ee74",
            "ReceiptHandle": "YjExOTYwMDUtZGVhZi00NGE5LWFhMWEtZWM0YjRkYjVmMWNjIGFybjphd3M6c3FzOnVzLWVhc3QtMTowMDAwMDAwMDAwMDA6cXVldWUtMiBkNDE3ZDE2Zi1iNjI3LTQzNGYtOTk5NC1hMjk3ZDZjNmVlNzQgMTcxMDE0NTk3Ni4xODM3NTEz",
            "MD5OfBody": "65a8e27d8879283831b664bd8b7f0ad4",
            "Body": "Hello, World!"
        }
    ]
}
```
Hooray! As you can see, the whole integration works well and our message was delivered to the queues, 
and we were able to receive it.

# Key Takeaways
This small guide allows you to simply configure LocalStack and the integration between SQS and SNS locally, 
so you can perform scenarios that are hard to perform in your test environments. 
Once you find out what options you want to add to the LocalStack configuration 
(e.g. debug log level or another AWS service), you can improve the configurations in our docker-compose.yml file 
and make testing even more comfortable.