---
title: Internal Endpoints
weight: 50
tags:
- internal-endpoints
- localstack-endpoints
description: >
  Overview of LocalStack and AWS specific internal endpoints for local development and testing
---

* uses
  * various local AWS services
  * LocalStack-specific features
* NOT part of the official AWS API
* available | paths
  * `/_localstack`
  * `/_aws` paths
* ways to access
  * [curl](https://curl.se/)
  * HTTP REST client
  * ...

## LocalStack endpoints

* `/_localstack`
  * API path -- for -- LocalStack internal resources

| Endpoint | Description |
| ------------------------------------ | --------------------------------------------------------------------------- |
| `/_localstack/health`                | check the available and running AWS services </br>  restart the LocalStack services  |
| `/_localstack/plugins`               | Shows the [Plux plugins](https://github.com/localstack/localstack/blob/master/docs/localstack-concepts/README.md#plugins) information in LocalStack. |
| `/_localstack/init`                  | Shows the initialization status after setting up [Init hooks](https://docs.localstack.cloud/references/init-hooks/). |
| `/_localstack/cloudformation/deploy` | Enables you to deploy CloudFormation templates locally through a web interface. |
| `/_localstack/diagnose`              | Reports extensive and sensitive data from LocalStack instance, enabled via the `DEBUG=1` configuration variable. |
| `/_localstack/config`                | Enables dynamic configuration updates at runtime, enabled via the `ENABLE_CONFIG_UPDATES` configuration variable.  |
| `/_localstack/chaos`                 | [Chaos API]({{< ref "chaos-api" >}}) configuration endpoint |
| `/_localstack/state/<service>/save`  | Get a snapshot of the given AWS service using the Persistence mechanism. |
| `/_localstack/state/<service>/load`  | Load the most recent snapshot of the given service using the Persistence mechanism. |
| `/_localstack/state/reset`           | Reset the state of the services using the Persistence mechanism. |
| `/_localstack/state/<service>/reset` | Reset the state of the given service using the Persistence mechanism. |

{{< callout "tip" >}}
You can use the `/_localstack/health` endpoint to restart or kill the services.
You can use [curl](https://curl.se/) or your HTTP REST client to access the endpoint:
{{< command >}}
$ curl -v --request POST --header "Content-Type: application/json"  --data '{"action":"restart"}' http://localhost:4566/_localstack/health
$ curl -v --request POST --header "Content-Type: application/json"  --data '{"action":"kill"}' http://localhost:4566/_localstack/health
{{< /command >}}
{{< /callout >}}

## AWS endpoints

The API path for the AWS internal resources is `/_aws`.
The following endpoints are available:

| Endpoint                               | Description                                               |
|----------------------------------------|-----------------------------------------------------------|
| `/_aws/lambda/runtimes`                | List Lambda runtimes. See [Lambda – Special Tools]({{< ref "user-guide/aws/lambda#special-tools" >}}) |
| `/_aws/sqs/messages`                   | Access all messages within a SQS queue                    |
| `/_aws/sns/platform-endpoint-messages` | Access and delete all the published SNS platform messages |
| `/_aws/ses`                            | Access and delete all the sent SES emails                 |
| `/_aws/cloudwatch/metrics/raw`         | Access all the raw CloudWatch metrics                     |
| `/_aws/cognito-idp`                    | Access the local Cognito login form                       |
| `/_aws/dynamodb/expired`               | Trigger the DynamoDB TTL worker at convenience            |
