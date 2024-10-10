---
title: "Filesystem Layout"
weight: 50
description: >
  Overview of runtime directory structure
aliases:
  - /localstack/filesystem/
---

* goal
  * filesystem directory layout / used internally -- by -- LocalStack
    * == created | LocalStack container
* filesystem layout
  * LocalStack v1+
  * if you want to disable it -> set `LEGACY_DIRECTORIES=1`
  * if you want to check it -> go into the container

```plaintext
/
├── etc
│   └── localstack
│       └── init
├── usr
│   └── lib
│       └── localstack
└── var
    └── lib
        └── localstack    // LocalStack volume directory root
            ├── cache
            ├── lib
            ├── logs
            ├── state
            └── tmp

```


## Directory contents

### LocalStack volume directory

* `/var/lib/localstack`
  * [LocalStack volume](#localstack-volume) directory root
* `/var/lib/localstack/lib`
  * variable packages 
    * == extensions or lazy-loaded third-party dependencies
* `/var/lib/localstack/logs`
  * logs for recent LocalStack runs
* `/var/lib/localstack/state`
  * == if persistence is enabled -> state of services (_Example:_ OpenSearch cluster data)
* `/var/lib/localstack/tmp`
  * == temporary data / NOT survive in LocalStack runs
    * == if LocalStack starts or stops -> it's cleared 
* `/var/lib/localstack/cache`
  * == temporary data / survive LocalStack runs  
    * == if LocalStack starts or stops -> it's NOT cleared

### Configuration

* `/etc/localstack`
  * configuration directory
* `/etc/localstack/init`
  * root directory for [initialization hooks]({{< ref `init-hooks` >}})
  * in future -> NOT needed
    * Reason: 🧠`/etc/localstack/conf.d` overrides🧠

### Static libraries

* `/usr/lib/localstack`
  * == static third-party packages / installed | container images
* `DATA_DIR`
  * deprecated LocalStack v1
    * Reason: 🧠directory convention is followed 🧠
    * if it's set -> value ignored
  * if `PERSISTENCE=1` (== persistence is enabled) -> `DATA_DIR` -- implicitly points to -- `/var/lib/localstack/state` 
* `HOST_TMP_DIR`
  * deprecated LocalStack v1
    * Reason: 🧠directory convention is followed 🧠
* `HOST_TMP_FOLDER`
  * == source of the bind mount to `/var/lib/localstack`

## LocalStack volume

* LocalStack volume -- must be mounted -- host:container's `/var/lib/localstack`
  * Reason: 🧠LocalStack works correctly 🧠 

### Using docker-compose

```yaml
volumes:
  - "${LOCALSTACK_VOLUME_DIR:-./volume}:/var/lib/localstack"
```

* 👁️`${LOCALSTACK_VOLUME_DIR}` could be an ARBITRARY location | host 👁️
  * _Example:_ `./volume`

    ```plaintext
    $ tree -L 4 ./volume
    .
    └── localstack
        ├── cache
        │   ├── machine.json
        │   ├── server.test.pem
        │   ├── server.test.pem.crt
        │   └── server.test.pem.key
        ├── lib
        │   └── opensearch
        │       └── 1.1.0
        ├── logs
        │   ├── localstack_infra.err
        │   └── localstack_infra.log
        ├── state
        │   └── startup_info.json
        └── tmp
            └── zipfile.4986fb95
    ```

### Using the CLI

* TODO:
When using the CLI to start LocalStack, the volume directory can be configured via the `LOCALSTACK_VOLUME_DIR`.
It should point to a directory on the host which is then automatically mounted into `/var/lib/localstack`.
The defaults are:

- Mac: `~/Library/Caches/localstack/volume`
- Linux: `~/.cache/localstack/volume`
- Windows: `%LOCALAPPDATA%\cache\localstack\volume`

## Host mode

When LocalStack is running in host mode, the system directories `/usr/lib/localstack` or `/var/lib/localstack` are not used.
Instead, the root directory is changed to `FILESYSTEM_ROOT` which defaults to `./.filesystem`, i.e. the LocalStack project root.

<!-- For further details, see https://github.com/localstack/localstack/pull/6302, https://github.com/localstack/localstack/pull/5011 -->
