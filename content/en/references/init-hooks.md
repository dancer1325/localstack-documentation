---
title: "Initialization Hooks"
weight: 50
description: >
  Writing shell or Python scripts to customize or initialize your LocalStack instance
aliases:
  - /localstack/init-hooks/
---

## Lifecycle stages and hooks

* 4 well-known lifecycle phases or stages
  * `BOOT`
    * container is running BUT LocalStack runtime has NOT been started
  * `START`
    * Python process is running & LocalStack runtime is starting
  * `READY`
    * LocalStack is ready to serve requests
  * `SHUTDOWN`
    * LocalStack is shutting down

* allows
  * hook into / lifecycle phases -- via -- custom
    * shell
    * Python scripts
  * mounting individual files, stage directories, or the entire init directory | your host -> into the container

* own directory | `/etc/localstack/init` / lifecycle phase

    ```plaintext
    /etc
    └── localstack
        └── init
            ├── boot.d           <-- executed in the container before localstack starts
            ├── ready.d          <-- executed when localstack becomes ready
            ├── shutdown.d       <-- executed when localstack shuts down
            └── start.d          <-- executed when localstack starts up
    ```
  * subdirectories can be created
  * uses
    * 👁️place your desired executable shell scripts (`.sh`) or Python programs (`.py`) | these directories 👁️
      * scripts or Python programs -- are executed from -- within a Python process
      * ALL scripts EXCEPT those | `boot.d` -- will be run in the -- SAME Python interpreter as LocalStack
        * -> additional ways of configuring/extending LocalStack
      * if you install `localstack-extension-terraform-init` [extension]({{<ref "user-guide/extensions/">}}) -> `.tf` files can also be supported
      * Shell scripts requirements
        * be executable
        * have a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) ( --usually -- `#!/bin/bash`)
      * script states
        * `UNKNOWN`
          * once script is discovered -> set in this state
        * `RUNNING`
        * `SUCCESSFUL`
        * `ERROR`

### Execution order and script failures

* Scripts 
  * sorted and executed | alphanumerical order
  * if they are placed | subdirectories -> scripts
    * | parent folders are executed first
    * directories -- are traversed in --
      * alphabetical order
      * depth first
  * if an init script fails -> remaining scripts will STILL be executed in order
  * requirements to consider a script as `ERROR`
    * shell script / returns non-zero exit code
    * Python script / raises an exception -- during its -- execution

## Status endpoint

* TODO:
There is an additional endpoint at `localhost:4566/_localstack/init` which returns the state of the initialization procedure.
Boot scripts (scripts placed in `boot.d`) are currently always in the `UNKNOWN` state, since they are executed outside the LocalStack process and we don't know whether they have been successfully executed or not.

```bash
curl -s localhost:4566/_localstack/init | jq .
```

```json
{
  "completed": {
    "BOOT": false,
    "START": true,
    "READY": true,
    "SHUTDOWN": false
  },
  "scripts": [
    {
      "stage": "BOOT",
      "name": "booting.sh",
      "state": "UNKNOWN"
    },
    {
      "stage": "READY",
      "name": "pre_seed.py",
      "state": "SUCCESSFUL"
    }
  ]
}
```

### Query individual stages

You can also query a specific stage at `localhost:4566/_localstack/init/<stage>`:

```bash
curl -s localhost:4566/_localstack/init/ready | jq .
```

```json
{
  "completed": true,
  "scripts": [
    {
      "stage": "READY",
      "name": "pre_seed.py",
      "state": "OK"
    }
  ]
}
```

To check whether a given stage has been completed you can now run, for example:

```bash
curl -s localhost:4566/_localstack/init/ready | jq .completed
```

which returns either `true` or `false`.

## Usage example

* pre-seeding LocalStack -- with -- custom simple state
  * if you have complex state -> check
    * [Cloud Pods]({{< ref "user-guide/state-management/cloud-pods" >}})
    * [how to auto-load them on startup]({{< ref "user-guide/state-management/cloud-pods#auto-loading-cloud-pods" >}})
  * _Example:_ create certain S3 bucket or DynamoDB table created | starting LocalStack
    * create a script `init-aws.sh` / 
      * `chmod +x init-aws.sh`
        * make it executable
      * mount it | `/etc/localstack/init/ready.d/`

    ```bash
    #!/bin/bash
    awslocal s3 mb s3://my-bucket
    awslocal sqs create-queue --queue-name my-queue
    ```

Start Localstack:

{{< tabpane >}}
{{< tab header="docker-compose.yml" lang="yml" >}}
version: "3.8"

services:
  localstack:
    container_name: "${LOCALSTACK_DOCKER_NAME:-localstack-main}"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
    environment:
      - DEBUG=1
    volumes:
      - "/path/to/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh"  # ready hook
      - "${LOCALSTACK_VOLUME_DIR:-./volume}:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
{{< /tab >}}
{{< tab header="CLI" lang="bash" >}}
# DOCKER_FLAGS are additional parameters to the `docker run` command of localstack start

DOCKER_FLAGS='-v /path/to/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh' localstack start
{{< /tab >}}
{{< /tabpane >}}

Another use for init hooks can be seen when [adding custom TLS certificates to LocalStack]({{< ref "custom-tls-certificates#custom-tls-certificates-with-init-hooks" >}}).

### Terraform configuration files as init hooks

Running Terraform configuration files as init hooks requires the installation of a special extension.
For more information on how to manage [LocalStack extensions]({{< ref "user-guide/extensions/" >}}), please refer to the dedicated documentation page,
and for more details on running init hooks in development mode, you can check out the [extension repository description](https://github.com/localstack/localstack-extensions/tree/main/terraform-init).

##### Usage

Start LocalStack with **`EXTENSION_AUTO_INSTALL="localstack-extension-terraform-init"`**.
Mount a **`main.tf`** file into **`/etc/localstack/init/ready.d`**
When LocalStack starts up, it will install the extension, which in turn installs Terraform and [`tflocal`](https://github.com/localstack/terraform-local) into the container.
If one of the init stage directories contain a `main.tf` file, the extension will run `tflocal init` and `tflocal apply` on that directory.

##### Example

main.tf:

```terraform
resource "aws_s3_bucket" "example" {
bucket = "my-tf-test-bucket"

tags = {
Name        = "My bucket"
Environment = "Dev"
  }
}
```

Start LocalStack Pro with mounted main.tf:

{{< tabpane >}}
{{< tab header="docker-compose.yml" lang="yml" >}}
version: "3.8"

services:
  localstack:
    container_name: "localstack-main"
    image: localstack/localstack-pro  # required for Pro
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
    environment:
# Activate LocalStack Pro: https://docs.localstack.cloud/getting-started/auth-token/
      - LOCALSTACK_AUTH_TOKEN=${LOCALSTACK_AUTH_TOKEN:?}
      - EXTENSION_AUTO_LOAD=localstack-extension-terraform-init
    volumes:
# you could also place your main.tf in `./ready.d` and set "./ready.d:/etc/localstack/init/ready.d"
      - "./main.tf:/etc/localstack/init/ready.d/main.tf"
      - "./volume:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
{{< /tab >}}

{{< tab header="LocalStack CLI" lang="bash" >}}
  localstack start \
  -e EXTENSION_AUTO_INSTALL="localstack-extension-terraform-init" \
  -v ./main.tf:/etc/localstack/init/ready.d/main.tf
{{< /tab >}}
{{< /tabpane >}}

You can wait for LocalStack to complete the startup process, and then print the created S3 bucket:
{{< command >}}
localstack wait && awslocal s3 ls
{{< / command >}}

The logs should show something like:

```bash
2024-06-26T20:36:19.946  INFO --- [ady_monitor)] l.extension                : Applying terraform project from file /etc/localstack/init/ready.d/main.tf
2024-06-26T20:36:19.946 DEBUG --- [ady_monitor)] localstack.utils.run       : Executing command: ['tflocal', '-chdir=/etc/localstack/init/ready.d', 'init', '-input=false']
2024-06-26T20:36:26.864 DEBUG --- [ady_monitor)] localstack.utils.run       : Executing command: ['tflocal', '-chdir=/etc/localstack/init/ready.d', 'apply', '-auto-approve']
```

For a more complex demo project, on how to use Terraform init hooks for your testing environments, you can check out [this example]({{< ref "tutorials/using-terraform-with-testcontainers-and-localstack/" >}}) in the Tutorials section.

## Troubleshooting

If you are having issues with your initialization hooks not being executed, please perform the following checks:
* Do the scripts have a known file extensions (`.sh` or `.py`)?
  * If not, rename the files to the matching file extension.
* Is the script file configured to use LF endings instead of CRLF endings?
  * If not, switch the file to LF mode as it is utilized in the Unix environment within a container.
* Is the script marked as executable?
  * If not, set the executable flag on the file (`chmod +x ...`).
* If it's a shell script, does it have a shebang (e.g., `#!/bin/bash`) as the first line in the file?
  * If not, add the shebang header (usually `#!/bin/bash`) on top of your script file.
* Is the script being listed in the logs when running LocalStack with `DEBUG=1`?
  * The detected scripts are logged like this:

    ```bash
    ...
    Init scripts discovered: {BOOT: [], START: [], READY: [Script(path='/etc/localstack/init/ready.d/setup.sh', stage=READY, state=UNKNOWN)], SHUTDOWN: []}
    ...
    Running READY script /etc/localstack/init/ready.d/setup.sh
    ...
    ```

  * If your script does not show up in the list of discovered init scripts, please check your Docker volume mount.
    Most likely the scripts are not properly mounted into the Docker container.
