---
title: Simulate AWS Services Locally with LocalStack
date: 2025-06-15T03:04:53+05:30
slug: localstack-getting-started
tags: ["localstack", "aws", "guide"]
categories: ["homelab", "TIL"]
---

Simulate AWS services on local machine for faster development and testing.

## What is LocalStack
In my day-to-day work, I need to interact with several AWS services (like S3, Route53, etc). When I am working locally, I do not have access to a full-fledged AWS account. Because doing so would incur additional costs and require constant internet access.

This is where [LocalStack](https://github.com/localstack/localstack) comes in. LocalStack is a drop-in replacement of AWS. Its a local AWS Cloud Stack, that allows us to mimic AWS services locally without needing a real AWS account or incurring any charges.

With LocalStack, we can run AWS services locally and test interactions with them without having any actual cloud setup and incurring charges.

## Running LocalStack with docker-compose
There are multiple ways to get started with LocalStack, I will be using docker-compose to start my LocalStack server. We can find more ways to install it in their [getting started](https://docs.localstack.cloud/getting-started/installation/) documentation.

Ensure Docker service is running and `docker-compose` is installed. Create this `docker-compose.yml` file locally:

```yaml
services:
  localstack:
    container_name: localstack-main
    image: localstack/localstack:4.5.0
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    env_file:
      - path: .env
        required: false
    environment:
      - DEBUG=${DEBUG:-0}
    volumes:
      - "localstack:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"

volumes:
  localstack:
```

Here, the container name is set to `localstack-main` and docker image for LocalStack is `4.5.0`. For latest tag, [check docker hub](https://hub.docker.com/r/localstack/localstack). The container will get a volume mounted at `/var/lib/localstack`. Mounting Docker socket `/var/run/docker.sock` as a volume is required for services that uses docker to provide the emulation, such as AWS Lambda.

Running `docker-compose up -d` will bring LocalStack up on `127.0.0.1:4566`

We can check the application info by accessing `http://localhost:4566/_localstack/info`:
```shell
$ curl -s http://localhost:4566/_localstack/info | jq
{
  "version": "4.5.0:0942fa747",
  "edition": "community",
  "is_license_activated": false,
  "session_id": "7ef52ddb-8cfc-4175-99d7-1a5c67a65bf5",
  "machine_id": "dkr_d9dc585e1065",
  "system": "linux",
  "is_docker": true,
  "server_time_utc": "2025-06-14T22:30:25",
  "uptime": 5203
}
```

To list all available AWS services, we can check by accessing `http://localhost:4566/_localstack/health`:
```shell
$ curl -s http://localhost:4566/_localstack/health | jq .
{
  "services": {
    "acm": "available",
    "apigateway": "available",
    "cloudformation": "available",
    "cloudwatch": "available",
    "config": "available",
    "dynamodb": "available",
    "dynamodbstreams": "available",
    "ec2": "available",
    "es": "available",
    "events": "available",
    "firehose": "available",
    "iam": "available",
    "kinesis": "available",
    "kms": "available",
    "lambda": "available",
    "logs": "available",
    "opensearch": "available",
    "redshift": "available",
    "resource-groups": "available",
    "resourcegroupstaggingapi": "available",
    "route53": "available",
    "route53resolver": "available",
    "s3": "running",
    "s3control": "available",
    "scheduler": "available",
    "secretsmanager": "available",
    "ses": "available",
    "sns": "available",
    "sqs": "available",
    "ssm": "available",
    "stepfunctions": "available",
    "sts": "running",
    "support": "available",
    "swf": "available",
    "transcribe": "available"
  },
  "edition": "community",
  "version": "4.5.0"
}
```

or in case [`localstack` CLI](https://docs.localstack.cloud/getting-started/installation/#installing-localstack-cli) is present, then by running `localstack status services`:
```shell
$ localstack status services
┏━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ Service                  ┃ Status      ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ acm                      │ ✔ available │
│ apigateway               │ ✔ available │
│ cloudformation           │ ✔ available │
│ cloudwatch               │ ✔ available │
│ config                   │ ✔ available │
│ dynamodb                 │ ✔ available │
│ dynamodbstreams          │ ✔ available │
│ ec2                      │ ✔ available │
│ es                       │ ✔ available │
│ events                   │ ✔ available │
│ firehose                 │ ✔ available │
│ iam                      │ ✔ available │
│ kinesis                  │ ✔ available │
│ kms                      │ ✔ available │
│ lambda                   │ ✔ available │
│ logs                     │ ✔ available │
│ opensearch               │ ✔ available │
│ redshift                 │ ✔ available │
│ resource-groups          │ ✔ available │
│ resourcegroupstaggingapi │ ✔ available │
│ route53                  │ ✔ available │
│ route53resolver          │ ✔ available │
│ s3                       │ ✔ running   │
│ s3control                │ ✔ available │
│ scheduler                │ ✔ available │
│ secretsmanager           │ ✔ available │
│ ses                      │ ✔ available │
│ sns                      │ ✔ available │
│ sqs                      │ ✔ available │
│ ssm                      │ ✔ available │
│ stepfunctions            │ ✔ available │
│ sts                      │ ✔ running   │
│ support                  │ ✔ available │
│ swf                      │ ✔ available │
│ transcribe               │ ✔ available │
└──────────────────────────┴─────────────┘
```

Once we have LocalStack setup and running, we can test AWS services locally. We would also need `awslocal` for testing these services. There are two ways to get LocalStack AWS CLI `awslocal`:
- Install it via pip. More information on this is [here](https://github.com/localstack/awscli-local)
- Create an alias.

I went with 2nd way as I was working on an Ubuntu Machine. This is the alias definition:
```shell
$ alias awslocal
alias awslocal='AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test aws --endpoint-url=http://127.0.0.1:4566'
```

Initially my alias was set as
```shell
alias awslocal="AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=us-east-1 aws --endpoint-url=http://127.0.0.1:4566"
```

But I kept encountering this error:
```
An error occurred (IllegalLocationConstraintException) when calling the CreateBucket operation: The unspecified location constraint is incompatible for the region specific endpoint this request was sent to.
```

So, I removed `AWS_DEFAULT_REGION=us-east-1`.

verifying that LocalStack is working:
```shell
$ aws configure
AWS Access Key ID [None]: test
AWS Secret Access Key [None]: test
Default region name [None]: us-east-1
Default output format [None]: json
$ awslocal sts get-caller-identity
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

If you see this, your LocalStack instance is working correctly!

## Example
Let's create an S3 bucket. Simply run `awslocal s3api create-bucket --bucket mytest`:
```shell
$ awslocal s3api create-bucket --bucket mytest
{
    "Location": "/mytest"
}
$ awslocal s3 ls
2025-06-15 04:21:59 mytest
$ echo "this is a test" > sample.txt
$ awslocal s3 cp sample.txt s3://mytest/sample.txt
upload: ./sample.txt to s3://mytest/sample.txt
$ awslocal s3 ls s3://mytest/
2025-06-15 04:25:14         15 sample.txt
$
```

Here we can see that I was able to create an S3 bucket and upload file to it.


## Conclusion
LocalStack is a powerful tool for anyone working with AWS. With just a Docker Compose file and a CLI alias, you can simulate a large portion of AWS locally, reducing your cloud bill and development friction.
