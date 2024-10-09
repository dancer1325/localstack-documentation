---
title: "AWS Command Line Interface (CLI)"
linkTitle: "AWS Command Line Interface (CLI)"
description: >
  Use AWS Command Line Interface (CLI) to create local AWS resources with LocalStack
---

## Introduction

* [AWS Command Line Interface (CLI)](https://aws.amazon.com/cli/)
  * == unified tool /
    * allows 
      * creating & managing AWS services -- via a -- CLI
    * uses
      * 👁️ALL CLI commands / -- applicable to -- services implemented | [LocalStack]({{< ref "references/coverage/" >}}) -> can be executed | operating against LocalStack 👁️
  * ways to use | LocalStack
    * own [AWS CLI]({{<ref "#aws-cli" >}})
    * [LocalStack AWS CLI]({{<ref "#localstack-aws-cli-awslocal">}})

## AWS CLI

* Follow 
  * [AWS CLI official installation](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  * [AWS CLI configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
    * _Example:_

    ```
    $ export AWS_ACCESS_KEY_ID="test"
    $ export AWS_SECRET_ACCESS_KEY="test"
    $ export AWS_DEFAULT_REGION="us-east-1"
    ```

* ways to configure the AWS CLI / AWS API requests -- are redirected to -- LocalStack
  * [Configuring an endpoint URL](#configuring-an-endpoint-url)
  * [Configuring a custom profile](#configuring-a-custom-profile)

### Configuring an endpoint URL

* _Example1:_ `aws --endpoint-url=http://localhost:4566 kinesis list-streams`
  * "localhost:4566" default port
* _Example2:_ if you want to enable the creation of pre-signed URLs / S3 buckets -> set  `AWS_ACCESS_KEY_ID="test"` & `AWS_SECRET_ACCESS_KEY="test"`
  * pre-signed URL signature verification algorithm -- validates --
    * pre-signed URL
    * its expiration

### Configuring a custom profile

* TODO:
You can configure a custom profile to use with LocalStack.
Add the following profile to your AWS configuration file (by default, this file is at `~/.aws/config`):

```bash
[profile localstack]
region=us-east-1
output=json
endpoint_url = http://localhost:4566
```

Add the following profile to your AWS credentials file (by default, this file is at `~/.aws/credentials`):

```bash
[localstack]
aws_access_key_id=test
aws_secret_access_key=test
```

You can now use the `localstack` profile with the `aws` CLI:

{{< command >}}
$ aws s3 mb s3://test --profile localstack
$ aws s3 ls --profile localstack
{{< / command >}}

{{< callout "tip" >}}
Alternatively, you can also set the `AWS_PROFILE=localstack` environment variable, in which case the `--profile localstack` parameter can be omitted in the commands above.
{{< /callout >}}

## LocalStack AWS CLI (`awslocal`)

`awslocal` serves as a thin wrapper and a substitute for the standard `aws` command, enabling you to run AWS CLI commands within the LocalStack environment without specifying the `--endpoint-url` parameter or a profile.

### Installation

Install the `awslocal` command using the following command:

{{< command >}}
$ pip install awscli-local[ver1]
{{< / command >}}

{{< callout "tip" >}}
The above command installs the most recent version of the underlying AWS CLI version 1 (`awscli`) package.
If you would rather manage your own `awscli` version (e.g., `v1` or `v2`) and only install the wrapper script, you can use the following command:

{{< command >}}
$ pip install awscli-local
{{< / command >}}
{{< /callout >}}

{{< callout >}}
Automatic installation of AWS CLI version 2 is currently not supported yet (at the time of writing there is no official pypi package for `v2` available), but the `awslocal` technically also works with AWS CLI v2 (see [this section]({{< ref "#limitations" >}}) for more details).
{{< /callout  >}}

### Usage

The `awslocal` command shares identical usage with the standard `aws` command.
For comprehensive usage instructions, refer to the manual pages by running `awslocal help`.

{{< command >}}
awslocal kinesis list-streams
{{< / command >}}

### Configuration

| Variable Name       | Description                                      |
|---------------------|--------------------------------------------------|
| AWS_ENDPOINT_URL    | The endpoint URL to connect to (takes precedence over USE_SSL/LOCALSTACK_HOST) |
| LOCALSTACK_HOST    | (deprecated) A variable defining where to find LocalStack (default: localhost:4566) |
| USE_SSL             | (deprecated) Whether to use SSL when connecting to LocalStack (default: False) |

### Current Limitations

Please note that there is a known limitation for using the `cloudformation package ...` command with the AWS CLI v2.
The problem is that the AWS CLI v2 is [not available as a package on pypi.org](https://github.com/aws/aws-cli/issues/4947), but is instead shipped as a binary package that cannot be easily patched from `awslocal`.
To work around this issue, you have 2 options:
* Downgrade to the v1 AWS CLI (this is the recommended approach)
* There is an unofficial way to install AWS CLI v2 from sources.
  We do not recommend this, but it is technically possible.
  Also, you should install these libraries in a Python virtualenv, to avoid version clashes with other libraries on your system:

{{< command >}}
$ virtualenv .venv
$ . .venv/bin/activate
$ pip install https://github.com/boto/botocore/archive/v2.zip https://github.com/aws/aws-cli/archive/v2.zip
{{< / command >}}

Please also note there is a known limitation for issuing requests using
`--no-sign-request` with the AWS CLI.
LocalStack's routing mechanism depends on
the signature of each request to identify the correct service for the request.
Thus, adding the flag `--no-sign-requests` provokes your request to reach the
wrong service.
One possible way to address this is to use the `awslocal` CLI
instead of AWS CLI.

## AWS CLI v2

Automatic installation of AWS CLI version 2 is currently not supported (at the time of writing there is no official pypi package for v2 available), but the awslocal technically also works with AWS CLI v2 (see this section for more details).

### AWS CLI v2 with Docker and LocalStack

By default, the container running [amazon/aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-docker.html) is isolated from `0.0.0.0:4566` on the host machine, that means that aws-cli cannot reach localstack through your shell.

To ensure that the two docker containers can communicate create a network on the docker engine:

{{< command >}}
$ docker network create localstack
0c9cb3d37b0ea1bfeb6b77ade0ce5525e33c7929d69f49c3e5ed0af457bdf123
{{< / command >}}

Then modify the `docker-compose.yml` specifying the network to use:

```yaml
networks:
  default:
    external:
      name: "localstack"
```

Run AWS Cli v2 docker container using this network (example):

{{< command >}}
$ docker run --network localstack --rm -it amazon/aws-cli --endpoint-url=http://localstack:4566 lambda list-functions
{
    "Functions": []
}
{{< / command >}}

If you use AWS CLI v2 from a docker container often, create an alias:

{{< command >}}
$ alias laws='docker run --network localstack --rm -it amazon/aws-cli --endpoint-url=http://localstack:4566'
{{< / command >}}

So you can type:

{{< command >}}
$ laws lambda list-functions
{
    "Functions": []
}
{{< / command >}}
