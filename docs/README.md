[![Build Status](https://travis-ci.org/broadinstitute/cromwell.svg?branch=develop)](https://travis-ci.org/broadinstitute/cromwell?branch=develop)
[![codecov](https://codecov.io/gh/broadinstitute/cromwell/branch/develop/graph/badge.svg)](https://codecov.io/gh/broadinstitute/cromwell)
[![Join the chat at https://gitter.im/broadinstitute/cromwell](https://badges.gitter.im/broadinstitute/cromwell.svg)](https://gitter.im/broadinstitute/cromwell?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Cromwell
========

A [Workflow Management System](https://en.wikipedia.org/wiki/Workflow_management_system) geared towards [scientific workflows](https://en.wikipedia.org/wiki/Scientific_workflow_system). Cromwell is open sourced under the BSD 3-Clause license.

# Runtime Attributes

Runtime attributes are used to customize tasks. Within a task one can specify runtime attributes to customize the environment for the call.

For example:

```
task jes_task {
  command {
    echo "Hello JES!"
  }
  runtime {
    docker: "ubuntu:latest"
    memory: "4G"
    cpu: "3"
    zones: "us-central1-c us-central1-b"
    disks: "/mnt/mnt1 3 SSD, /mnt/mnt2 500 HDD"
  }
}
workflow jes_workflow {
  call jes_task
}
```

This table lists the currently available runtime attributes for cromwell:

| Runtime Attribute    | LOCAL |  JES  |  SGE  |
| -------------------- |:-----:|:-----:|:-----:|
| continueOnReturnCode |   x   |   x   |   x   |
| cpu                  |       |   x   |   x   |
| disks                |       |   x   |       |
| zones                |       |   x   |       |
| docker               |   x   |   x   |   x   |
| failOnStderr         |   x   |   x   |   x   |
| memory               |       |   x   |   x   |
| preemptible          |       |   x   |       |
| bootDiskSizeGb       |       |   x   |       |

Runtime attribute values are interpreted as expressions.  This means that it is possible to express the value of a runtime attribute as a function of one of the task's inputs.  For example:

```
task runtime_test {
  String ubuntu_tag
  Int memory_gb

  command {
    ./my_binary
  }

  runtime {
    docker: "ubuntu:" + ubuntu_tag
    memory: memory_gb + "GB"
  }
}
```

SGE and similar backends may define other configurable runtime attributes beyond the five listed. See [Sun GridEngine](#sun-gridengine-backend) for more information.

## Specifying Default Values

Default values for runtime attributes can be specified via [workflow options](#workflow-options).  For example, consider this WDL file:

```wdl
task first {
  command { ... }
}

task second {
  command {...}
  runtime {
    docker: "my_docker_image"
  }
}

workflow w {
  call first
  call second
}
```

And this set of workflow options:

```json
{
  "default_runtime_attributes": {
    "docker": "ubuntu:latest",
    "zones": "us-central1-c us-central1-b"
  }
}
```

Then these values for `docker` and `zones` will be used for any task that does not explicitly override them in the WDL file. So the effective runtime for `task first` is:
```
{
    "docker": "ubuntu:latest",
    "zones": "us-central1-c us-central1-b"
  }
```
And the effective runtime for `task second` is:
```
{
    "docker": "my_docker_image",
    "zones": "us-central1-c us-central1-b"
  }
```
Note how for task second, the WDL value for `docker` is used instead of the default provided in the workflow options.

## continueOnReturnCode

When each task finishes it returns a code. Normally, a non-zero return code indicates a failure. However you can override this behavior by specifying the `continueOnReturnCode` attribute.

When set to false, any non-zero return code will be considered a failure. When set to true, all return codes will be considered successful.

```
runtime {
  continueOnReturnCode: true
}
```

When set to an integer, or an array of integers, only those integers will be considered as successful return codes.

```
runtime {
  continueOnReturnCode: 1
}
```

```
runtime {
  continueOnReturnCode: [0, 1]
}
```

Defaults to "0".

## cpu

Passed to JES: "The minimum number of cores to use."

Passed to SGE, etc.: Configurable, but usually a reservation and/or limit of number of cores.

```
runtime {
  cpu: 2
}
```

Defaults to "1".

## disks

Passed to JES: "Disks to attach."

The disks are specified as a comma separated list of disks. Each disk is further separated as a space separated triplet of:

1. Mount point (absolute path), or `local-disk` to reference the mount point where JES will localize files and the task's current working directory will be
2. Disk size in GB (ignored for disk type LOCAL)
3. Disk type.  One of: "LOCAL", "SSD", or "HDD" ([documentation](https://cloud.google.com/compute/docs/disks/#overview))

All tasks launched on JES *must* have a `local-disk`.  If one is not specified in the runtime section of the task, then a default of `local-disk 10 SSD` will be used.  The `local-disk` will be mounted to `/cromwell_root`.

The Disk type must be one of "LOCAL", "SSD", or "HDD". When set to "LOCAL", the size of the drive is automatically provisioned by Google so any size specified in WDL will be ignored. All disks are set to auto-delete after the job completes.

**Example 1: Changing the Localization Disk**

```
runtime {
  disks: "local-disk 100 SSD"
}
```

**Example 2: Mounting an Additional Two Disks**

```
runtime {
  disks: "/mnt/my_mnt 3 SSD, /mnt/my_mnt2 500 HDD"
}
```

### Boot Disk
In addition to working disks, JES allows specification of a boot disk size. This is the disk where the docker image itself is booted, **not the working directory of your task on the VM**.
Its primary purpose is to ensure that larger docker images can fit on the boot disk.
```
runtime {
  # Yikes, we have a big OS in this docker image! Allow 50GB to hold it:
  bootDiskSizeGb: 50
}
```

Since no `local-disk` entry is specified, Cromwell will automatically add `local-disk 10 SSD` to this list.

## zones

The ordered list of zone preference (see [Region and Zones](https://cloud.google.com/compute/docs/zones) documentation for specifics)

The zones are specified as a space separated list, with no commas.

```
runtime {
  zones: "us-central1-c us-central1-b"
}
```

Defaults to the configuration setting `genomics.default-zones` in the JES configuration block which in turn defaults to using `us-central1-b`

## docker

When specified, cromwell will run your task within the specified Docker image.

```
runtime {
  docker: "ubuntu:latest"
}
```

This attribute is mandatory when submitting tasks to JES. When running on other backends, they default to not running the process within Docker.

## failOnStderr

Some programs write to the standard error stream when there is an error, but still return a zero exit code. Set `failOnStderr` to true for these tasks, and it will be considered a failure if anything is written to the standard error stream.

```
runtime {
  failOnStderr: true
}
```

Defaults to "false".

## memory

Passed to JES: "The minimum amount of RAM to use."

Passed to SGE, etc.: Configurable, but usually a reservation and/or limit of memory.

The memory size is specified as an amount and units of memory, for example "4 G".

```
runtime {
  memory: "4G"
}
```

Defaults to "2G".

## preemptible

Passed to JES: "If applicable, preemptible machines may be used for the run."

Take an Int as a value that indicates the maximum number of times Cromwell should request a preemptible machine for this task before defaulting back to a non-preemptible one.
eg. With a value of 1, Cromwell will request a preemptible VM, if the VM is preempted, the task will be retried with a non-preemptible VM.
Note: If specified, this attribute overrides [workflow options](#workflow-options).

```
runtime {
  preemptible: 1
}
```

Defaults to 0.

# Logging

Cromwell accepts two Java Properties or Environment Variables for controlling logging:

* `LOG_MODE` - Accepts either `pretty` or `standard` (default `pretty`).  In `standard` mode, logs will be written without ANSI escape code coloring, with a layout more appropriate for server logs, versus `pretty` that is easier to read for a single workflow run.
* `LOG_LEVEL` - Level at which to log (default `info`).

Additionally, a directory may be set for writing per workflow logs. By default, the per workflow logs will be erased once the workflow completes.

```hocon
// In application.conf or specified via system properties
workflow-options {
    workflow-log-dir: "cromwell-workflow-logs"
    workflow-log-temporary: true
}
```

The usual case of generating the temporary per workflow logs is to copy them to a remote directory, while deleting the local copy to preserve local disk space. To specify the remote directory to copy the logs to use the separate [workflow option](#workflow-options) `final_workflow_log_dir`.

Cromwell supports [Sentry](https://docs.sentry.io), a service that can be used to monitor exceptions reported in an application's logs. To make use of this add `-Dsentry.dsn=DSN_URL` to your Java command line with your DSN URL.

# Instrumentation

## StatsD

Cromwell collects metrics while it's running and sends them to an internal service. By default this service ignores those metrics, but it can be configured to forward them to a [StatsD](https://github.com/etsy/statsd) server.
To do so, simply add this snippet to your configuration file:

```hocon
services.Instrumentation.class = "cromwell.services.instrumentation.impl.akka.AkkaInstrumentationServiceActor"
```
Make sure to configure your StatsD service:

```hocon
services.Instrumentation.config.statsd {
    hostname = "localhost" # Replace with your host
    port = 8125 # Replace with your port
    # prefix = "my_prefix" # All metrics will be prefixed by this value if present.
    flush-rate = 1 second # Rate at which metrics are sent to the StatsD server
  }
```

There is also an additional configuration value that can be set: 

```hocon
# Rate at which Cromwell updates its gauge values (number of workflows running, queued, etc...)
system.instrumentation-rate = 5 seconds
```

### Metrics

The current StatsD implementation uses metrics-statsd to report instrumentation values.
metrics-statsd reports all metrics with a gauge type.
This means all the metric will be under the gauge section. We might add /remove metrics in the future depending on need and usage.
However here the current high level categories:

`backend`, `rest-api`, `job`, `workflow`, `io`

# Workflow Options

When running a workflow from the [command line](#run) or [REST API](#post-apiworkflowsversion), one may specify a JSON file that toggles various options for running the workflow.  From the command line, the workflow options is passed in as the third positional parameter to the 'run' subcommand.  From the REST API, it's an optional part in the multi-part POST request.  See the respective sections for more details.

Example workflow options file:

```json
{
  "jes_gcs_root": "gs://my-bucket/workflows",
  "google_project": "my_google_project",
  "refresh_token": "1/Fjf8gfJr5fdfNf9dk26fdn23FDm4x"
}
```

Valid keys and their meanings:

* Global *(use with any backend)*
    * **write_to_cache** - Accepts values `true` or `false`.  If `false`, the completed calls from this workflow will not be added to the cache.  See the [Call Caching](#call-caching) section for more details.
    * **read_from_cache** - Accepts values `true` or `false`.  If `false`, Cromwell will not search the cache when invoking a call (i.e. every call will be executed unconditionally).  See the [Call Caching](#call-caching) section for more details.
    * **final_workflow_log_dir** - Specifies a path where per-workflow logs will be written.  If this is not specified, per-workflow logs will not be copied out of the Cromwell workflow log temporary directory/path before they are deleted.
    * **final_workflow_outputs_dir** - Specifies a path where final workflow outputs will be written.  If this is not specified, workflow outputs will not be copied out of the Cromwell workflow execution directory/path.
    * **final_call_logs_dir** - Specifies a path where final call logs will be written.  If this is not specified, call logs will not be copied out of the Cromwell workflow execution directory/path.
    * **default_runtime_attributes** - A JSON object where the keys are [runtime attributes](#runtime-attributes) and the values are defaults that will be used through the workflow invocation.  Individual tasks can choose to override these values.  See the [runtime attributes](#specifying-default-values) section for more information.
    * **continueOnReturnCode** - Can accept a boolean value or a comma separated list of integers in a string.  Defaults to false.  If false, then only return code of 0 will be acceptable for a task invocation.  If true, then any return code is valid.  If the value is a list of comma-separated integers in a string, this is interpreted as the acceptable return codes for this task.
    * **workflow_failure_mode** - What happens after a task fails. Choose from:
        * **ContinueWhilePossible** - continues to start and process calls in the workflow, as long as they did not depend on the failing call
        * **NoNewCalls** - no *new* calls are started but existing calls are allowed to finish
        * The default is `NoNewCalls` but this can be changed using the `workflow-options.workflow-failure-mode` configuration option.
    * **backend** - Override the default backend specified in the Cromwell configuration for this workflow only.
* JES Backend Only
    * **jes_gcs_root** - (JES backend only) Specifies where outputs of the workflow will be written.  Expects this to be a GCS URL (e.g. `gs://my-bucket/workflows`).  If this is not set, this defaults to the value within `backend.jes.config.root` in the [configuration](#configuring-cromwell).
    * **google_compute_service_account** - (JES backend only) Specifies an alternate service account to use on the compute instance (e.g. my-new-svcacct@my-google-project.iam.gserviceaccount.com).  If this is not set, this defaults to the value within `backend.jes.config.genomics.compute-service-account` in the [configuration](#configuring-cromwell) if specified or `default` otherwise.
    * **google_project** - (JES backend only) Specifies which google project to execute this workflow.
    * **refresh_token** - (JES backend only) Only used if `localizeWithRefreshToken` is specified in the [configuration file](#configuring-cromwell).
    * **auth_bucket** - (JES backend only) defaults to the the value in **jes_gcs_root**.  This should represent a GCS URL that only Cromwell can write to.  The Cromwell account is determined by the `google.authScheme` (and the corresponding `google.userAuth` and `google.serviceAuth`)
    * **monitoring_script** - (JES backend only) Specifies a GCS URL to a script that will be invoked prior to the user command being run.  For example, if the value for monitoring_script is "gs://bucket/script.sh", it will be invoked as `./script.sh > monitoring.log &`.  The value `monitoring.log` file will be automatically de-localized.

# Labels

Every call run on the JES backend is given certain labels by default, so that Google resources can be queried by these labels later. The current default label set automatically applied is:

| Key | Value | Example | Notes |
|-----|-------|---------|-------|
| cromwell-workflow-id | The Cromwell ID given to the root workflow (i.e. the ID returned by Cromwell on submission) | cromwell-d4b412c5-bf3d-4169-91b0-1b635ce47a26 | To fit the required [format](#label-format), we prefix with 'cromwell-' |
| cromwell-sub-workflow-name | The name of this job's sub-workflow | my-sub-workflow | Only present if the task is called in a subworkflow. |
| wdl-task-name | The name of the WDL task | my-task | |
| wdl-call-alias | The alias of the WDL call that created this job | my-task-1 | Only present if the task was called with an alias. |

## Custom Labels File

Custom labels can also be applied to every call in a workflow by specifying a custom labels file when the workflow is submitted. This file should be in JSON format and contain a set of fields: `"label-key": "label-value" `. For example:
```
{
  "label-key-1": "label-value-1",
  "label-key-2": "label-value-2",
  "label-key-3": "label-value-3"
}
```

## Label Format

When labels are supplied to Cromwell, it will fail any request containing invalid label strings. Below are the requirements for a valid label key/value pair in Cromwell:
- Label keys and values can't contain characters other than `[a-z]`, `[0-9]` or `-`.
- Label keys must start with `[a-z]` and end with `[a-z]` or `[0-9]`.
- Label values must start and end with `[a-z]` or `[0-9]`.
- Label keys may not be empty but label values may be empty.
- Label key and values have a max char limit of 63.

Google has a different schema for labels, where label key and value strings must match the regex `[a-z]([-a-z0-9]*[a-z0-9])?` and be no more than 63 characters in length.
For automatically applied labels, Cromwell will modify workflow/task/call names to fit the schema, according to the following rules:
- Any capital letters are lowercased.
- Any character which is not one of `[a-z]`, `[0-9]` or `-` will be replaced with `-`.
- If the start character does not match `[a-z]` then prefix with `x--`
- If the final character does not match `[a-z0-9]` then suffix with `--x`
- If the string is too long, only take the first 30 and last 30 characters and add `---` between them.

# Call Caching

Call Caching allows Cromwell to detect when a job has been run in the past so it doesn't have to re-compute results.  Cromwell searches the cache of previously run jobs for a one that has the exact same command and exact same inputs.  If a previously run job is found in the cache, Cromwell will **copy the results** of the previous job instead of re-running it.

Cromwell's call cache is maintained in its database.  For best mileage with call caching, configure Cromwell to [point to a MySQL database](#database) instead of the default in-memory database.  This way any invocation of Cromwell (either with `run` or `server` subcommands) will be able to utilize results from all calls that are in that database.

**Call Caching is disabled by default.**  Once enabled, Cromwell will search the call cache for every `call` statement invocation, assuming `read_from_cache` is enabled (see below):

* If there was no cache hit, the `call` will be executed as normal.  Once finished it will add itself to the cache, assuming `read_from_cache` is enabled (see below)
* If there was a cache hit, outputs are **copied** from the cached job to the new job's output directory

> **Note:** If call caching is enabled, be careful not to change the contents of the output directory for any previously run job.  Doing so might cause cache hits in Cromwell to copy over modified data and Cromwell currently does not check that the contents of the output directory changed.

## Configuring Call Caching
To enable Call Caching, add the following to your Cromwell [configuration](#configuring-cromwell):

```
call-caching {
  enabled = true
  invalidate-bad-cache-results = true
}
```

When `call-caching.enabled=true` (default: `false`), Cromwell will be able to to copy results from previously run jobs (when appropriate).
When `invalidate-bad-cache-results=true` (default: `true`), Cromwell will invalidate any cache results which fail to copy during a cache-hit. This is usually desired but might be unwanted if a cache might fail to copy for external reasons, such as a difference in user authentication.

## Call Caching Workflow Options
Cromwell also accepts two [workflow option](#workflow-options) related to call caching:

* If call caching is enabled, but one wishes to run a workflow but not add any of the calls into the call cache when they finish, the `write_to_cache` option can be set to `false`.  This value defaults to `true`.
* If call caching is enabled, but you don't want to check the cache for any `call` invocations, set the option `read_from_cache` to `false`.  This value also defaults to `true`

> **Note:** If call caching is disabled, the workflow options `read_from_cache` and `write_to_cache` will be ignored and the options will be treated as though they were 'false'.

## Docker Tags

Docker tags are a convenient way to point to a version of an image (ubuntu:14.04), or even the latest version (ubuntu:latest).
For that purpose, tags are mutable, meaning that the image they point to can change, while the tag name stays the same.
While this is very convenient in some cases, using mutable, or "floating" tags in tasks affects the reproducibility of a workflow: the same workflow using "ubuntu:latest" run now, and a year, or even a month from now will actually run with different docker images.
This has an even bigger impact when Call Caching is turned on in Cromwell, and could lead to unpredictable behaviors if a tag is updated in the middle of a workflow or even a scatter for example.
Docker provides another way of identifying an image version, using the specific digest of the image. The digest is guaranteed to be different if 2 images have different byte content. For more information see https://docs.docker.com/registry/spec/api/#/content-digests
A docker image with digest can be referenced as follows : **ubuntu@sha256:71cd81252a3563a03ad8daee81047b62ab5d892ebbfbf71cf53415f29c130950**
The above image refers to a specific image of ubuntu, that does not depend on a floating tag.
A workflow containing this Docker image run now and a year from now will run in the exact same container.

In order to remove unpredictable behaviors, Cromwell takes the following approach regarding floating docker tags.

When Cromwell finds a job ready to be run, it will first look at its docker runtime attribute, and apply the following logic:

* The job doesn't specify a docker image: The job will be dispatched and all call caching settings (read/write) will apply normally.
* The job does specify a docker runtime attribute:
    * The docker image uses a hash: All call caching settings apply normally
    * The docker image uses a floating tag:
        * Cromwell will attempt to look up the hash of the image. Upon success it will pass both the floating tag and this hash value to the backend.
        * All backends currently included with Cromwell will utilize this hash value to run the job.
        * Within a single workflow all floating tags will resolve to the same hash value even if Cromwell is restarted when the workflow is running.
        * If Cromwell fails to lookup the hash (unsupported registry, wrong credentials, ...) it will run the job with the user provided floating tag.
        * The actual Docker image (floating tag or hash) used for the job will be reported in the `dockerImageUsed` attribute of the call metadata.

### Docker Lookup

Cromwell provides 2 methods to lookup a docker hash from a docker tag:

* Local
    In this mode, cromwell will first attempt to find the image on the local machine where it's running using the docker CLI. If the image is present, then its hash will be used.
    If it's not present, cromwell will execute a `docker pull` to try and retrieve it. If this succeeds, the newly retrieved hash will be used. Otherwise the lookup will be considered failed.
    Note that cromwell runs the `docker` CLI the same way a human would. This means two things:
     * The machine Cromwell is running on needs to have docker installed and a docker daemon running.
     * Whichever credentials (and only those) are available on that machine will be available to pull the image.
    
* Remote
    In this mode, cromwell will attempt to retrieve the hash by contacting the remote docker registry where the image is stored. This currently supports Docker Hub and GCR.
    
    Docker registry and access levels supported by Cromwell for docker hash lookup in "remote" mode:
    
    |       |       DockerHub    ||       GCR       ||
    |:-----:|:---------:|:-------:|:------:|:-------:|
    |       |   Public  | Private | Public | Private |
    |  JES  |     X     |    X    |    X   |    X    |
    | Other |     X     |         |    X   |         |

## Local Filesystem Options
When running a job on the Config (Shared Filesystem) backend, Cromwell provides some additional options in the backend's config section:

```
      config {
        ...
        filesystems {
          ...
          local {
            ...
            caching {
              # When copying a cached result, what type of file duplication should occur. Attempted in the order listed below:
              duplication-strategy: [
                "hard-link", "soft-link", "copy"
              ]

              # Possible values: file, path
              # "file" will compute an md5 hash of the file content.
              # "path" will compute an md5 hash of the file path. This strategy will only be effective if the duplication-strategy (above) is set to "soft-link",
              # in order to allow for the original file path to be hashed.
              # Default: file
              hashing-strategy: "file"

              # When true, will check if a sibling file with the same name and the .md5 extension exists, and if it does, use the content of this file as a hash.
              # If false or the md5 does not exist, will proceed with the above-defined hashing strategy.
              # Default: false
              check-sibling-md5: false
            }
          }
        }
      }
```
# Imports

Import statements inside of a workflow file are supported by Cromwell when running in Server mode as well as Single Workflow Runner Mode.

In Single Workflow Runner Mode, you pass in a zip file which includes the WDL files referenced by the import statements. Cromwell requires the zip file to be passed in as a command line argument, as explained by the section [run](#run).

For example, given a workflow `wf.wdl` and an imports directory `WdlImports.zip`, a sample command would be:
```
java -jar cromwell.jar wf.wdl wf.inputs - - WdlImports.zip
```

In Server Mode, you pass in a zip file using the parameter `workflowDependencies` via the [POST /api/workflows/:version](#post-apiworkflowsversion) endpoint.


# Sub Workflows

WDL allows the execution of an entire workflow as a step in a larger workflow (see WDL SPEC for more details), which is what will be referred to as a sub workflow going forward.
Cromwell supports execution of such workflows. Note that sub workflows can themselves contain sub workflows, etc... There is no limitation as to how deeply workflows can be nested.

## Execution

Sub workflows are executed exactly as a task would be.
*This means that if another call depends on an output of a sub workflow, this call will run when the whole sub workflow completes (successfully).*
For example, in the following case :

`main.wdl`
```
import "sub_wdl.wdl" as sub

workflow main_workflow {

    call sub.hello_and_goodbye { input: hello_and_goodbye_input = "sub world" }
    
    # call myTask { input: hello_and_goodbye.hello_output }
    
    output {
        String main_output = hello_and_goodbye.hello_output
    }
}
```

`sub_wdl.wdl`
```
task hello {
  String addressee
  command {
    echo "Hello ${addressee}!"
  }
  runtime {
      docker: "ubuntu:latest"
  }
  output {
    String salutation = read_string(stdout())
  }
}

task goodbye {
  String addressee
  command {
    echo "Goodbye ${addressee}!"
  }
  runtime {
      docker: "ubuntu:latest"
  }
  output {
    String salutation = read_string(stdout())
  }
}

workflow hello_and_goodbye {
  String hello_and_goodbye_input
  
  call hello {input: addressee = hello_and_goodbye_input }
  call goodbye {input: addressee = hello_and_goodbye_input }
  
  output {
    String hello_output = hello.salutation
    String goodbye_output = goodbye.salutation
  }
}
```

`myTask` will start only when hello_and_goodbye completes (which means all of its calls are done), even though `myTask` only needs the output of hello in the hello_and_goodbye sub workflow. 
If hello_and_goodbye fails, then `myTask` won't be executed.
Only workflow outputs are visible outside a workflow, which means that references to outputs produced by a sub workflow will only be valid if those outputs are exposed in the workflow output section.

Sub workflows are executed in the context of a main workflow, which means that operations that are normally executed once per workflow (set up, clean up, outputs copying, log copying, etc...)
will NOT be re-executed for each sub workflow. For instance if a resource is created during workflow initialization, sub workflows will need to share this same resource.
Workflow outputs will be copied for the main root workflow but not for intermediate sub workflows.

Restarts, aborts, and call-caching work exactly as they would with tasks. 
All tasks run by a sub workflow are eligible for call caching under the same rules as any other task.
However, workflows themselves are not cached as such. Which means that running the exact same workflow twice with call caching on will trigger each task to cache individually,
but not the workflow itself.

The root path for sub workflow execution files (scripts, output files, logs) will be under the parent workflow call directory.
For example, the execution directory for the above main workflow would look like the following:

```
cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/ <- main workflow id
└── call-hello_and_goodbye <- call directory for call hello_and_goodbye in the main workflow
    └── hello_and_goodbye <- name of the sub workflow 
        └── a6365f91-c807-465a-9186-a5d3da98fe11 <- sub workflow id
            ├── call-goodbye
            │   └── execution
            │       ├── rc
            │       ├── script
            │       ├── script.background
            │       ├── script.submit
            │       ├── stderr
            │       ├── stderr.background
            │       ├── stdout
            │       └── stdout.background
            └── call-hello
                └── execution
                    ├── rc
                    ├── script
                    ├── script.background
                    ├── script.submit
                    ├── stderr
                    ├── stderr.background
                    ├── stdout
                    └── stdout.background

```

## Metadata
Each sub workflow will have its own workflow ID. This ID will appear in the metadata of the parent workflow, in the call section corresponding to the sub workflow, under the "subWorkflowId" attribute.
For example, querying the `main_workflow` metadata above (minus the `myTask` call) , could result in something like this:

`GET /api/workflows/v2/1d919bd4-d046-43b0-9918-9964509689dd/metadata`

```
{
  "workflowName": "main_workflow",
  "submittedFiles": {
    "inputs": "{}",
    "workflow": "import \"sub_wdl.wdl\" as sub\n\nworkflow main_workflow {\n\n    call sub.hello_and_goodbye { input: hello_and_goodbye_input = \"sub world\" }\n    \n    # call myTask { input: hello_and_goodbye.hello_output }\n    \n    output {\n        String main_output = hello_and_goodbye.hello_output\n    }\n}",
    "options": "{\n\n}"
  },
  "calls": {
    "main_workflow.hello_and_goodbye": [
      {
        "executionStatus": "Done",
        "shardIndex": -1,
        "outputs": {
          "goodbye_output": "Goodbye sub world!",
          "hello_output": "Hello sub world!"
        },
        "inputs": {
          "hello_and_goodbye_input": "sub world"
        },
        "end": "2016-11-17T14:13:41.117-05:00",
        "attempt": 1,
        "start": "2016-11-17T14:13:39.236-05:00",
        "subWorkflowId": "a6365f91-c807-465a-9186-a5d3da98fe11"
      }
    ]
  },
  "outputs": {
    "main_output": "Hello sub world!"
  },
  "workflowRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd",
  "id": "1d919bd4-d046-43b0-9918-9964509689dd",
  "inputs": {},
  "submission": "2016-11-17T14:13:39.104-05:00",
  "status": "Succeeded",
  "end": "2016-11-17T14:13:41.120-05:00",
  "start": "2016-11-17T14:13:39.204-05:00"
}
```

The sub workflow ID can be queried separately:

`GET /api/workflows/v2/a6365f91-c807-465a-9186-a5d3da98fe11/metadata`

```
{
  "workflowName": "hello_and_goodbye",
  "calls": {
    "sub.hello_and_goodbye.hello": [
      {
        "executionStatus": "Done",
        "stdout": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello/execution/stdout",
        "shardIndex": -1,
        "outputs": {
          "salutation": "Hello sub world!"
        },
        "runtimeAttributes": {
          "docker": "ubuntu:latest",
          "failOnStderr": false,
          "continueOnReturnCode": "0"
        },
        "cache": {
          "allowResultReuse": true
        },
        "Effective call caching mode": "CallCachingOff",
        "inputs": {
          "addressee": "sub world"
        },
        "returnCode": 0,
        "jobId": "49830",
        "backend": "Local",
        "end": "2016-11-17T14:13:40.712-05:00",
        "stderr": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello/execution/stderr",
        "callRoot": "/cromwell/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello",
        "attempt": 1,
        "executionEvents": [
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "Pending",
            "endTime": "2016-11-17T14:13:39.240-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "RequestingExecutionToken",
            "endTime": "2016-11-17T14:13:39.240-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "PreparingJob",
            "endTime": "2016-11-17T14:13:39.243-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.243-05:00",
            "description": "RunningJob",
            "endTime": "2016-11-17T14:13:40.704-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:40.704-05:00",
            "description": "UpdatingJobStore",
            "endTime": "2016-11-17T14:13:40.712-05:00"
          }
        ],
        "start": "2016-11-17T14:13:39.239-05:00"
      }
    ],
    "sub.hello_and_goodbye.goodbye": [
      {
        "executionStatus": "Done",
        "stdout": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye/execution/stdout",
        "shardIndex": -1,
        "outputs": {
          "salutation": "Goodbye sub world!"
        },
        "runtimeAttributes": {
          "docker": "ubuntu:latest",
          "failOnStderr": false,
          "continueOnReturnCode": "0"
        },
        "cache": {
          "allowResultReuse": true
        },
        "Effective call caching mode": "CallCachingOff",
        "inputs": {
          "addressee": "sub world"
        },
        "returnCode": 0,
        "jobId": "49831",
        "backend": "Local",
        "end": "2016-11-17T14:13:41.115-05:00",
        "stderr": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye/execution/stderr",
        "callRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye",
        "attempt": 1,
        "executionEvents": [
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "Pending",
            "endTime": "2016-11-17T14:13:39.240-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "RequestingExecutionToken",
            "endTime": "2016-11-17T14:13:39.240-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.240-05:00",
            "description": "PreparingJob",
            "endTime": "2016-11-17T14:13:39.243-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:39.243-05:00",
            "description": "RunningJob",
            "endTime": "2016-11-17T14:13:41.112-05:00"
          },
          {
            "startTime": "2016-11-17T14:13:41.112-05:00",
            "description": "UpdatingJobStore",
            "endTime": "2016-11-17T14:13:41.115-05:00"
          }
        ],
        "start": "2016-11-17T14:13:39.239-05:00"
      }
    ]
  },
  "outputs": {
    "goodbye_output": "Goodbye sub world!",
    "hello_output": "Hello sub world!"
  },
  "workflowRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11",
  "id": "a6365f91-c807-465a-9186-a5d3da98fe11",
  "inputs": {
    "hello_and_goodbye_input": "sub world"
  },
  "status": "Succeeded",
  "parentWorkflowId": "1d919bd4-d046-43b0-9918-9964509689dd",
  "end": "2016-11-17T14:13:41.116-05:00",
  "start": "2016-11-17T14:13:39.236-05:00"
}
```

It's also possible to set the URL query parameter `expandSubWorkflows` to `true` to automatically include sub workflows metadata (`false` by default).

`GET api/workflows/v2/1d919bd4-d046-43b0-9918-9964509689dd/metadata?expandSubWorkflows=true`

```
{
  "workflowName": "main_workflow",
  "submittedFiles": {
    "inputs": "{}",
    "workflow": "import \"sub_wdl.wdl\" as sub\n\nworkflow main_workflow {\n\n    call sub.hello_and_goodbye { input: hello_and_goodbye_input = \"sub world\" }\n    \n    # call myTask { input: hello_and_goodbye.hello_output }\n    \n    output {\n        String main_output = hello_and_goodbye.hello_output\n    }\n}",
    "options": "{\n\n}"
  },
  "calls": {
    "main_workflow.hello_and_goodbye": [{
      "executionStatus": "Done",
      "subWorkflowMetadata": {
        "workflowName": "hello_and_goodbye",
        "calls": {
          "sub.hello_and_goodbye.hello": [{
            "executionStatus": "Done",
            "stdout": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello/execution/stdout",
            "shardIndex": -1,
            "outputs": {
              "salutation": "Hello sub world!"
            },
            "runtimeAttributes": {
              "docker": "ubuntu:latest",
              "failOnStderr": false,
              "continueOnReturnCode": "0"
            },
            "cache": {
              "allowResultReuse": true
            },
            "Effective call caching mode": "CallCachingOff",
            "inputs": {
              "addressee": "sub world"
            },
            "returnCode": 0,
            "jobId": "49830",
            "backend": "Local",
            "end": "2016-11-17T14:13:40.712-05:00",
            "stderr": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello/execution/stderr",
            "callRoot": "cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-hello",
            "attempt": 1,
            "executionEvents": [{
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "Pending",
              "endTime": "2016-11-17T14:13:39.240-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "RequestingExecutionToken",
              "endTime": "2016-11-17T14:13:39.240-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "PreparingJob",
              "endTime": "2016-11-17T14:13:39.243-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.243-05:00",
              "description": "RunningJob",
              "endTime": "2016-11-17T14:13:40.704-05:00"
            }, {
              "startTime": "2016-11-17T14:13:40.704-05:00",
              "description": "UpdatingJobStore",
              "endTime": "2016-11-17T14:13:40.712-05:00"
            }],
            "start": "2016-11-17T14:13:39.239-05:00"
          }],
          "sub.hello_and_goodbye.goodbye": [{
            "executionStatus": "Done",
            "stdout": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye/execution/stdout",
            "shardIndex": -1,
            "outputs": {
              "salutation": "Goodbye sub world!"
            },
            "runtimeAttributes": {
              "docker": "ubuntu:latest",
              "failOnStderr": false,
              "continueOnReturnCode": "0"
            },
            "cache": {
              "allowResultReuse": true
            },
            "Effective call caching mode": "CallCachingOff",
            "inputs": {
              "addressee": "sub world"
            },
            "returnCode": 0,
            "jobId": "49831",
            "backend": "Local",
            "end": "2016-11-17T14:13:41.115-05:00",
            "stderr": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye/execution/stderr",
            "callRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11/call-goodbye",
            "attempt": 1,
            "executionEvents": [{
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "Pending",
              "endTime": "2016-11-17T14:13:39.240-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "RequestingExecutionToken",
              "endTime": "2016-11-17T14:13:39.240-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.240-05:00",
              "description": "PreparingJob",
              "endTime": "2016-11-17T14:13:39.243-05:00"
            }, {
              "startTime": "2016-11-17T14:13:39.243-05:00",
              "description": "RunningJob",
              "endTime": "2016-11-17T14:13:41.112-05:00"
            }, {
              "startTime": "2016-11-17T14:13:41.112-05:00",
              "description": "UpdatingJobStore",
              "endTime": "2016-11-17T14:13:41.115-05:00"
            }],
            "start": "2016-11-17T14:13:39.239-05:00"
          }]
        },
        "outputs": {
          "goodbye_output": "Goodbye sub world!",
          "hello_output": "Hello sub world!"
        },
        "workflowRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd/call-hello_and_goodbye/hello_and_goodbye/a6365f91-c807-465a-9186-a5d3da98fe11",
        "id": "a6365f91-c807-465a-9186-a5d3da98fe11",
        "inputs": {
          "hello_and_goodbye_input": "sub world"
        },
        "status": "Succeeded",
        "parentWorkflowId": "1d919bd4-d046-43b0-9918-9964509689dd",
        "end": "2016-11-17T14:13:41.116-05:00",
        "start": "2016-11-17T14:13:39.236-05:00"
      },
      "shardIndex": -1,
      "outputs": {
        "goodbye_output": "Goodbye sub world!",
        "hello_output": "Hello sub world!"
      },
      "inputs": {
        "hello_and_goodbye_input": "sub world"
      },
      "end": "2016-11-17T14:13:41.117-05:00",
      "attempt": 1,
      "start": "2016-11-17T14:13:39.236-05:00"
    }]
  },
  "outputs": {
    "main_output": "Hello sub world!"
  },
  "workflowRoot": "/cromwell-executions/main_workflow/1d919bd4-d046-43b0-9918-9964509689dd",
  "id": "1d919bd4-d046-43b0-9918-9964509689dd",
  "inputs": {

  },
  "submission": "2016-11-17T14:13:39.104-05:00",
  "status": "Succeeded",
  "end": "2016-11-17T14:13:41.120-05:00",
  "start": "2016-11-17T14:13:39.204-05:00"
}
```

# REST API

The `server` subcommand on the executable JAR will start an HTTP server which can accept workflow files to run as well as check status and output of existing workflows.

The following sub-sections define which HTTP Requests the web server can accept and what they will return.  Example HTTP requests are given in [HTTPie](https://github.com/jkbrzt/httpie) and [cURL](https://curl.haxx.se/)

## REST API Versions

All web server requests include an API version in the url. The current version is `v1`.

## POST /api/workflows/:version

This endpoint accepts a POST request with a `multipart/form-data` encoded body.  The form fields that may be included are:

* `workflowSource` - *Required* Contains the workflow source file to submit for execution.
* `workflowType` - *Optional* The type of the `workflowSource`.
   If not specified, returns the `workflow-options.default.workflow-type` configuration value with a default of `WDL`.
* `workflowTypeVersion` - *Optional* The version of the `workflowType`, for example "draft-2".
   If not specified, returns the `workflow-options.default.workflow-type-version` configuration value with no default.
* `workflowInputs` - *Optional* JSON file containing the inputs.  For WDL workflows a skeleton file can be generated from [wdltool](https://github.com/broadinstitute/wdltool) using the "inputs" subcommand.
* `workflowInputs_n` - *Optional* Where `n` is an integer. JSON file containing the 'n'th set of auxiliary inputs.
* `workflowOptions` - *Optional* JSON file containing options for this workflow execution.  See the [run](#run) CLI sub-command for some more information about this.
* `customLabels` - *Optional* JSON file containing a set of custom labels to apply to this workflow. See [Labels](#labels) for the expected format.
* `workflowDependencies` - *Optional* ZIP file containing workflow source files that are used to resolve import statements.

Regarding the workflowInputs parameter, in case of key conflicts between multiple input JSON files, higher values of x in workflowInputs_x override lower values. For example, an input specified in workflowInputs_3 will override an input with the same name in workflowInputs or workflowInputs_2.
Similarly, an input key specified in workflowInputs_5 will override an identical input key in any other input file.

Additionally, although Swagger has a limit of 5 JSON input files, the REST endpoint itself can accept an unlimited number of JSON input files.


cURL:

```
$ curl -v "localhost:8000/api/workflows/v1" -F workflowSource=@src/main/resources/3step.wdl -F workflowInputs=@test.json
```

HTTPie:

```
$ http --print=hbHB --form POST localhost:8000/api/workflows/v1 workflowSource=@src/main/resources/3step.wdl workflowInputs@inputs.json
```

Request:

```
POST /api/workflows/v1 HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 730
Content-Type: multipart/form-data; boundary=64128d499e9e4616adea7d281f695dca
Host: localhost:8000
User-Agent: HTTPie/0.9.2

--64128d499e9e4616adea7d281f695dca
Content-Disposition: form-data; name="workflowSource"

task ps {
  command {
    ps
  }
  output {
    File procs = stdout()
  }
}

task cgrep {
  command {
    grep '${pattern}' ${File in_file} | wc -l
  }
  output {
    Int count = read_int(stdout())
  }
}

task wc {
  command {
    cat ${File in_file} | wc -l
  }
  output {
    Int count = read_int(stdout())
  }
}

workflow three_step {
  call ps
  call cgrep {
    input: in_file=ps.procs
  }
  call wc {
    input: in_file=ps.procs
  }
}

--64128d499e9e4616adea7d281f695dca
Content-Disposition: form-data; name="workflowInputs"; filename="inputs.json"

{
    "three_step.cgrep.pattern": "..."
}

--64128d499e9e4616adea7d281f695dca--
```

Response:

```
HTTP/1.1 201 Created
Content-Length: 74
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 18:06:28 GMT
Server: spray-can/1.3.3

{
    "id": "69d1d92f-3895-4a7b-880a-82535e9a096e",
    "status": "Submitted"
}
```

To specify workflow options as well:


cURL:

```
$ curl -v "localhost:8000/api/workflows/v1" -F workflowSource=@wdl/jes0.wdl -F workflowInputs=@wdl/jes0.json -F workflowOptions=@options.json
```

HTTPie:

```
http --print=HBhb --form POST http://localhost:8000/api/workflows/v1 workflowSource=@wdl/jes0.wdl workflowInputs@wdl/jes0.json workflowOptions@options.json
```

Request (some parts truncated for brevity):

```
POST /api/workflows/v1 HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 1472
Content-Type: multipart/form-data; boundary=f3fd038395644de596c460257626edd7
Host: localhost:8000
User-Agent: HTTPie/0.9.2

--f3fd038395644de596c460257626edd7
Content-Disposition: form-data; name="workflowSource"

task x { ... }
task y { ... }
task z { ... }

workflow myworkflow {
  call x
  call y
  call z {
    input: example="gs://my-bucket/cromwell-executions/myworkflow/example.txt", int=3000
  }
}

--f3fd038395644de596c460257626edd7
Content-Disposition: form-data; name="workflowInputs"; filename="jes0.json"

{
  "myworkflow.x.x": "100"
}

--f3fd038395644de596c460257626edd7
Content-Disposition: form-data; name="workflowOptions"; filename="options.json"

{
  "jes_gcs_root": "gs://myworkflow-dev/workflows"
}

--f3fd038395644de596c460257626edd7--
```

## POST /api/workflows/:version/batch

This endpoint accepts a POST request with a `multipart/form-data`
encoded body.  The form fields that may be included are:

* `workflowSource` - *Required* Contains the workflow source file to submit for
execution.
* `workflowInputs` - *Required* JSON file containing the inputs in a
JSON array. For WDL workflows a skeleton file for a single inputs json element can be
generated from [wdltool](https://github.com/broadinstitute/wdltool)
using the "inputs" subcommand. The orderded endpoint responses will
contain one workflow submission response for each input, respectively.
* `workflowOptions` - *Optional* JSON file containing options for this
workflow execution.  See the [run](#run) CLI sub-command for some more
information about this.
* `workflowDependencies` - *Optional* ZIP file containing workflow source files that are used to resolve import statements. Applied equally to all workflowInput sets.

cURL:

```
$ curl -v "localhost:8000/api/workflows/v1/batch" -F workflowSource=@src/main/resources/3step.wdl -F workflowInputs=@test_array.json
```

HTTPie:

```
$ http --print=hbHB --form POST localhost:8000/api/workflows/v1/batch workflowSource=@src/main/resources/3step.wdl workflowInputs@inputs_array.json
```

Request:

```
POST /api/workflows/v1/batch HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 750
Content-Type: multipart/form-data; boundary=64128d499e9e4616adea7d281f695dcb
Host: localhost:8000
User-Agent: HTTPie/0.9.2

--64128d499e9e4616adea7d281f695dcb
Content-Disposition: form-data; name="workflowSource"

task ps {
  command {
    ps
  }
  output {
    File procs = stdout()
  }
}

task cgrep {
  command {
    grep '${pattern}' ${File in_file} | wc -l
  }
  output {
    Int count = read_int(stdout())
  }
}

task wc {
  command {
    cat ${File in_file} | wc -l
  }
  output {
    Int count = read_int(stdout())
  }
}

workflow three_step {
  call ps
  call cgrep {
    input: in_file=ps.procs
  }
  call wc {
    input: in_file=ps.procs
  }
}

--64128d499e9e4616adea7d281f695dcb
Content-Disposition: form-data; name="workflowInputs"; filename="inputs_array.json"

[
    {
        "three_step.cgrep.pattern": "..."
    },
    {
        "three_step.cgrep.pattern": "..."
    }
]

--64128d499e9e4616adea7d281f695dcb--
```

Response:

```
HTTP/1.1 201 Created
Content-Length: 96
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 18:06:28 GMT
Server: spray-can/1.3.3

[
    {
        "id": "69d1d92f-3895-4a7b-880a-82535e9a096e",
        "status": "Submitted"
    },
    {
        "id": "69d1d92f-3895-4a7b-880a-82535e9a096f",
        "status": "Submitted"
    }
]
```

To specify workflow options as well:


cURL:

```
$ curl -v "localhost:8000/api/workflows/v1/batch" -F workflowSource=@wdl/jes0.wdl -F workflowInputs=@wdl/jes0_array.json -F workflowOptions=@options.json
```

HTTPie:

```
http --print=HBhb --form POST http://localhost:8000/api/workflows/v1/batch workflowSource=@wdl/jes0.wdl workflowInputs@wdl/jes0_array.json workflowOptions@options.json
```

Request (some parts truncated for brevity):

```
POST /api/workflows/v1/batch HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 1492
Content-Type: multipart/form-data; boundary=f3fd038395644de596c460257626edd8
Host: localhost:8000
User-Agent: HTTPie/0.9.2

--f3fd038395644de596c460257626edd8
Content-Disposition: form-data; name="workflowSource"

task x { ... }
task y { ... }
task z { ... }

workflow myworkflow {
  call x
  call y
  call z {
    input: example="gs://my-bucket/cromwell-executions/myworkflow/example.txt", int=3000
  }
}

--f3fd038395644de596c460257626edd8
Content-Disposition: form-data; name="workflowInputs"; filename="jes0_array.json"

[
  {
    "myworkflow.x.x": "100"
  }, {
    "myworkflow.x.x": "101"
  }
]

--f3fd038395644de596c460257626edd8
Content-Disposition: form-data; name="workflowOptions"; filename="options.json"

{
  "jes_gcs_root": "gs://myworkflow-dev/workflows"
}

--f3fd038395644de596c460257626edd8--
```


## GET /api/workflows/:version/query

This endpoint allows for querying workflows based on the following criteria:

* `name`
* `id`
* `status`
* `label`
* `start` (start datetime with mandatory offset)
* `end` (end datetime with mandatory offset)
* `page` (page of results)
* `pagesize` (# of results per page)

Names, ids, and statuses can be given multiple times to include
workflows with any of the specified names, ids, or statuses. When
multiple names are specified, any workflow matching one of the names
will be returned. The same is true for multiple ids or statuses. When
more than one label is specified, only workflows associated to all of
the given labels will be returned. 

When a combination of criteria are specified, for example querying by 
names and statuses, the results must return workflows that match one of 
the specified names and one of the statuses. Using page and pagesize will
enable server side pagination.

Valid statuses are `Submitted`, `Running`, `Aborting`, `Aborted`, `Failed`, and `Succeeded`.  `start` and `end` should
be in [ISO8601 datetime](https://en.wikipedia.org/wiki/ISO_8601) format with *mandatory offset* and `start` cannot be after `end`.

cURL:

```
$ curl "http://localhost:8000/api/workflows/v1/query?start=2015-11-01T00%3A00%3A00-04%3A00&end=2015-11-04T00%3A00%3A00-04%3A00&status=Failed&status=Succeeded&page=1&pagesize=10"
```

HTTPie:

```
$ http "http://localhost:8000/api/workflows/v1/query?start=2015-11-01T00%3A00%3A00-04%3A00&end=2015-11-04T00%3A00%3A00-04%3A00&status=Failed&status=Succeeded&page=1&pagesize=10"
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 133
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 18:06:56 GMT
Server: spray-can/1.3.3

{
  "results": [
    {
      "name": "w",
      "id": "fdfa8482-e870-4528-b639-73514b0469b2",
      "status": "Succeeded",
      "end": "2015-11-01T07:45:52.000-05:00",
      "start": "2015-11-01T07:38:57.000-05:00"
    },
    {
      "name": "hello",
      "id": "e69895b1-42ed-40e1-b42d-888532c49a0f",
      "status": "Succeeded",
      "end": "2015-11-01T07:45:30.000-05:00",
      "start": "2015-11-01T07:38:58.000-05:00"
    },
    {
      "name": "crasher",
      "id": "ed44cce4-d21b-4c42-b76d-9d145e4d3607",
      "status": "Failed",
      "end": "2015-11-01T07:45:44.000-05:00",
      "start": "2015-11-01T07:38:59.000-05:00"
    }
  ]
}
```

Labels have to be queried in key and value pairs separated by a colon, i.e. `label-key:label-value`. For example, if a batch of workflows was submitted with the following labels JSON:
```
{
  "label-key-1" : "label-value-1",
  "label-key-2" : "label-value-2"
}
```

A request to query for succeeded workflows with both labels would be:

cURL:
```
$ curl "http://localhost:8000/api/workflows/v1/query?status=Succeeded&label=label-key-1:label-value-1&label=label-key-2:label-value-2
```

HTTPie:
```
$ http "http://localhost:8000/api/workflows/v1/query?status=Succeeded&label=label-key-1:label-value-1&label=label-key-2:label-value-2
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 608
Content-Type: application/json; charset=UTF-8
Date: Tue, 9 May 2017 20:24:33 GMT
Server: spray-can/1.3.3

{
    "results": [
        {
            "end": "2017-05-09T16:07:30.515-04:00", 
            "id": "83fc23d5-48d1-456e-997a-087e55cd2e06", 
            "name": "wf_hello", 
            "start": "2017-05-09T16:01:51.940-04:00", 
            "status": "Succeeded"
        }, 
        {
            "end": "2017-05-09T16:07:13.174-04:00", 
            "id": "7620a5c6-a5c6-466c-994b-dd8dca917b9b", 
            "name": "wf_goodbye", 
            "start": "2017-05-09T16:01:51.939-04:00", 
            "status": "Succeeded"
        }
    ]
}
```

Query data is refreshed from raw data periodically according to the configuration value `services.MetadataService.metadata-summary-refresh-interval`.
This interval represents the duration between the end of one summary refresh sweep and the beginning of the next sweep.  If not specified the
refresh interval will default to 2 seconds.  To turn off metadata summary refresh, specify an infinite refresh interval value with "Inf".

```
services {
  MetadataService {
    config {
      metadata-summary-refresh-interval = "10 seconds"
    }
  }
}
```

## POST /api/workflows/:version/query

This endpoint allows for querying workflows based on the same criteria
as [GET /api/workflows/:version/query](#get-apiworkflowsversionquery).

Instead of specifying query parameters in the URL, the parameters
must be sent via the POST body. The request content type must be
`application/json`. The json should be a list of objects. Each json
object should contain a different criterion.

cURL:

```
$ curl -X POST --header "Content-Type: application/json" -d "[{\"start\": \"2015-11-01T00:00:00-04:00\"}, {\"end\": \"2015-11-04T00:00:00-04:00\"}, {\"status\": \"Failed\"}, {\"status\": \"Succeeded\"}, {\"page\": \"1\"}, {\"pagesize\": \"10\"}]" "http://localhost:8000/api/workflows/v1/query"
```

HTTPie:

```
$ echo "[{\"start\": \"2015-11-01T00:00:00-04:00\"}, {\"end\": \"2015-11-04T00:00:00-04:00\"}, {\"status\": \"Failed\"}, {\"status\": \"Succeeded\"}, {\"page\": \"1\"}, {\"pagesize\": \"10\"}]" | http "http://localhost:8000/api/workflows/v1/query"
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 133
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 18:06:56 GMT
Server: spray-can/1.3.3

{
  "results": [
    {
      "name": "w",
      "id": "fdfa8482-e870-4528-b639-73514b0469b2",
      "status": "Succeeded",
      "end": "2015-11-01T07:45:52.000-05:00",
      "start": "2015-11-01T07:38:57.000-05:00"
    },
    {
      "name": "hello",
      "id": "e69895b1-42ed-40e1-b42d-888532c49a0f",
      "status": "Succeeded",
      "end": "2015-11-01T07:45:30.000-05:00",
      "start": "2015-11-01T07:38:58.000-05:00"
    },
    {
      "name": "crasher",
      "id": "ed44cce4-d21b-4c42-b76d-9d145e4d3607",
      "status": "Failed",
      "end": "2015-11-01T07:45:44.000-05:00",
      "start": "2015-11-01T07:38:59.000-05:00"
    }
  ]
}
```

## PATCH /api/workflows/:version/:id/labels

This endpoint is used to update multiple labels for an existing workflow. When supplying a label with a key unique to the workflow submission, a new label key/value entry is appended to that workflow's metadata. When supplying a label with a key that is already associated to the workflow submission, the original label value is updated with the new value for that workflow's metadata.

The [labels](#labels) must be a mapping of key/value pairs in JSON format that are sent via the PATCH body. The request content type must be
`application/json`.

cURL:

```
$ curl -X PATCH --header "Content-Type: application/json" -d "{\"label-key-1\":\"label-value-1\", \"label-key-2\": \"label-value-2\"}" "http://localhost:8000/api/workflows/v1/c4c6339c-8cc9-47fb-acc5-b5cb8d2809f5/labels"
```

HTTPie:

```
$ echo '{"label-key-1":"label-value-1", "label-key-2": "label-value-2"}' | http PATCH "http://localhost:8000/api/workflows/v1/c4c6339c-8cc9-47fb-acc5-b5cb8d2809f5/labels"
```

Response:
```
{ "id": "c4c6339c-8cc9-47fb-acc5-b5cb8d2809f5",
  "labels":
    {
      "label-key-1": "label-value-1",
      "label-key-2": "label-value-2"
    }
}
```


## GET /api/workflows/:version/:id/status

cURL:

```
$ curl http://localhost:8000/api/workflows/v1/69d1d92f-3895-4a7b-880a-82535e9a096e/status
```

HTTPie:

```
$ http http://localhost:8000/api/workflows/v1/69d1d92f-3895-4a7b-880a-82535e9a096e/status
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 74
Content-Type: application/json; charset=UTF-8
Date: Tue, 02 Jun 2015 18:06:56 GMT
Server: spray-can/1.3.3

{
    "id": "69d1d92f-3895-4a7b-880a-82535e9a096e",
    "status": "Succeeded"
}
```

## GET /api/workflows/:version/:id/outputs

cURL:

```
$ curl http://localhost:8000/api/workflows/v1/e442e52a-9de1-47f0-8b4f-e6e565008cf1/outputs
```

HTTPie:

```
$ http http://localhost:8000/api/workflows/v1/e442e52a-9de1-47f0-8b4f-e6e565008cf1/outputs
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 241
Content-Type: application/json; charset=UTF-8
Date: Thu, 04 Jun 2015 12:15:33 GMT
Server: spray-can/1.3.3

{
    "id": "e442e52a-9de1-47f0-8b4f-e6e565008cf1",
    "outputs": {
        "three_step.cgrep.count": 8,
        "three_step.ps.procs": "/var/folders/kg/c7vgxnn902lc3qvc2z2g81s89xhzdz/T/stdout2814345504446060277.tmp",
        "three_step.wc.count": 8
    }
}
```

## GET /api/workflows/:version/:id/timing

This endpoint is meant to be used in a web browser.  It will show a Gantt Chart of a particular workflow.  The bars in the chart represent start and end times for individual task invocations.

![Timing diagram](http://i.imgur.com/EOE2HoL.png)

## GET /api/workflows/:version/:id/logs

This will return paths to the standard out and standard error files that were generated during the execution of all calls in a workflow.

A call has one or more standard out and standard error logs, depending on if the call was scattered or not. In the latter case, one log is provided for each instance of the call that has been run.

cURL:

```
$ curl http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/logs
```

HTTPie:

```
$ http http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/logs
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 379
Content-Type: application/json; charset=UTF-8
Date: Mon, 03 Aug 2015 17:11:28 GMT
Server: spray-can/1.3.3

{
    "id": "b3e45584-9450-4e73-9523-fc3ccf749848",
    "logs": {
        "call.ps": [
            {
                "stderr": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-ps/stderr6126967977036995110.tmp",
                "stdout": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-ps/stdout6128485235785447571.tmp"
            }
        ],
        "call.cgrep": [
            {
                "stderr": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-cgrep/stderr6126967977036995110.tmp",
                "stdout": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-cgrep/stdout6128485235785447571.tmp"
            }
        ],
        "call.wc": [
            {
                "stderr": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-wc/stderr6126967977036995110.tmp",
                "stdout": "/home/user/test/b3e45584-9450-4e73-9523-fc3ccf749848/call-wc/stdout6128485235785447571.tmp"
            }
        ]
    }
}
```

## GET /api/workflows/:version/:id/metadata

This endpoint returns a superset of the data from #get-workflowsversionidlogs in essentially the same format
(i.e. shards are accounted for by an array of maps, in the same order as the shards). 
In addition to shards, every attempt that was made for this call will have its own object as well, in the same order as the attempts.
Workflow metadata includes submission, start, and end datetimes, as well as status, inputs and outputs.
Call-level metadata includes inputs, outputs, start and end datetime, backend-specific job id,
return code, stdout and stderr.  Date formats are ISO with milliseconds.

Accepted parameters are:

* `includeKey` Optional repeated string value, specifies what metadata
  keys to include in the output, matched as a prefix string. Keys that
  are not specified are filtered out. The call keys `attempt` and
  `shardIndex` will always be included. May not be used with
  `excludeKey`.

* `excludeKey` Optional repeated string value, specifies what metadata
  keys to exclude from the output, matched as a prefix string. Keys that
  are specified are filtered out. The call keys `attempt` and
  `shardIndex` will always be included. May not be used with
  `includeKey`.

cURL:

```
$ curl http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/metadata
```

HTTPie:

```
$ http http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/metadata
```

Response:
```
HTTP/1.1 200 OK
Server spray-can/1.3.3 is not blacklisted
Server: spray-can/1.3.3
Date: Thu, 01 Oct 2015 22:18:07 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 7286
{
  "workflowName": "sc_test",
  "submittedFiles": {
      "inputs": "{}",
      "workflow": "task do_prepare {\n    File input_file\n    command {\n        split -l 1 ${input_file} temp_ && ls -1 temp_?? > files.list\n    }\n    output {\n        Array[File] split_files = read_lines(\"files.list\")\n    }\n}\n# count the number of words in the input file, writing the count to an output file overkill in this case, but simulates a real scatter-gather that would just return an Int (map)\ntask do_scatter {\n    File input_file\n    command {\n        wc -w ${input_file} > output.txt\n    }\n    output {\n        File count_file = \"output.txt\"\n    }\n}\n# aggregate the results back together (reduce)\ntask do_gather {\n    Array[File] input_files\n    command <<<\n        cat ${sep = ' ' input_files} | awk '{s+=$$1} END {print s}'\n    >>>\n    output {\n        Int sum = read_int(stdout())\n    }\n}\nworkflow sc_test {\n    call do_prepare\n    scatter(f in do_prepare.split_files) {\n        call do_scatter {\n            input: input_file = f\n        }\n    }\n    call do_gather {\n        input: input_files = do_scatter.count_file\n    }\n}",
      "options": "{\n\n}",
      "workflowType": "WDL"
  },
  "calls": {
    "sc_test.do_prepare": [
      {
        "executionStatus": "Done",
        "stdout": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/stdout",
        "backendStatus": "Done",
        "shardIndex": -1,
        "outputs": {
          "split_files": [
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_aa",
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_ad"
          ]
        },
        "inputs": {
          "input_file": "/home/jdoe/cromwell/11.txt"
        },
        "runtimeAttributes": {
            "failOnStderr": "true",
            "continueOnReturnCode": "0"
        },
        "callCaching": {
            "allowResultReuse": true,
            "hit": false,
            "result": "Cache Miss",
            "hashes": {
              "output count": "C4CA4238A0B923820DCC509A6F75849B",
              "runtime attribute": {
                "docker": "N/A",
                "continueOnReturnCode": "CFCD208495D565EF66E7DFF9F98764DA",
                "failOnStderr": "68934A3E9455FA72420237EB05902327"
              },
              "output expression": {
                "Array": "D856082E6599CF6EC9F7F42013A2EC4C"
              },
              "input count": "C4CA4238A0B923820DCC509A6F75849B",
              "backend name": "509820290D57F333403F490DDE7316F4",
              "command template": "9F5F1F24810FACDF917906BA4EBA807D",
              "input": {
                "File input_file": "11fa6d7ed15b42f2f73a455bf5864b49"
              }
            },
            "effectiveCallCachingMode": "ReadAndWriteCache"
        },
        "jobId": "34479",
        "returnCode": 0,
        "backend": "Local",
        "end": "2016-02-04T13:47:56.000-05:00",
        "stderr": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/stderr",
        "attempt": 1,
        "executionEvents": [],
        "start": "2016-02-04T13:47:55.000-05:00"
      }
    ],
    "sc_test.do_scatter": [
      {
        "executionStatus": "Preempted",
        "stdout": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/stdout",
        "backendStatus": "Preempted",
        "shardIndex": 0,
        "outputs": {},
        "runtimeAttributes": {
           "failOnStderr": "true",
           "continueOnReturnCode": "0"
        },
        "callCaching": {
          "allowResultReuse": true,
          "hit": false,
          "result": "Cache Miss",
          "hashes": {
            "output count": "C4CA4238A0B923820DCC509A6F75849B",
            "runtime attribute": {
              "docker": "N/A",
              "continueOnReturnCode": "CFCD208495D565EF66E7DFF9F98764DA",
              "failOnStderr": "68934A3E9455FA72420237EB05902327"
            },
            "output expression": {
              "File count_file": "EF1B47FFA9990E8D058D177073939DF7"
            },
            "input count": "C4CA4238A0B923820DCC509A6F75849B",
            "backend name": "509820290D57F333403F490DDE7316F4",
            "command template": "FD00A1B0AB6A0C97B0737C83F179DDE7",
            "input": {
              "File input_file": "a53794d214dc5dedbcecdf827bf683a2"
            }
          },
         "effectiveCallCachingMode": "ReadAndWriteCache"
        },
        "inputs": {
          "input_file": "f"
        },
        "jobId": "34496",
        "backend": "Local",
        "end": "2016-02-04T13:47:56.000-05:00",
        "stderr": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/stderr",
        "attempt": 1,
        "executionEvents": [],
        "start": "2016-02-04T13:47:56.000-05:00"
      },
      {
        "executionStatus": "Done",
        "stdout": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/stdout",
        "backendStatus": "Done",
        "shardIndex": 0,
        "outputs": {
          "count_file": "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt"
        },
        "runtimeAttributes": {
           "failOnStderr": "true",
           "continueOnReturnCode": "0"
        },
        "callCaching": {
          "allowResultReuse": true,
          "hit": false,
          "result": "Cache Miss",
          "hashes": {
            "output count": "C4CA4238A0B923820DCC509A6F75849B",
            "runtime attribute": {
              "docker": "N/A",
              "continueOnReturnCode": "CFCD208495D565EF66E7DFF9F98764DA",
              "failOnStderr": "68934A3E9455FA72420237EB05902327"
            },
            "output expression": {
              "File count_file": "EF1B47FFA9990E8D058D177073939DF7"
            },
            "input count": "C4CA4238A0B923820DCC509A6F75849B",
            "backend name": "509820290D57F333403F490DDE7316F4",
            "command template": "FD00A1B0AB6A0C97B0737C83F179DDE7",
            "input": {
              "File input_file": "a53794d214dc5dedbcecdf827bf683a2"
            }
          },
         "effectiveCallCachingMode": "ReadAndWriteCache"
        },
        "inputs": {
          "input_file": "f"
        },
        "returnCode": 0,
        "jobId": "34965",
        "end": "2016-02-04T13:47:56.000-05:00",
        "stderr": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/stderr",
        "attempt": 2,
        "executionEvents": [],
        "start": "2016-02-04T13:47:56.000-05:00"
      },
      {
        "executionStatus": "Done",
        "stdout": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/stdout",
        "backendStatus": "Done",
        "shardIndex": 1,
        "outputs": {
          "count_file": "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
        },
        "runtimeAttributes": {
           "failOnStderr": "true",
           "continueOnReturnCode": "0"
        },
        "callCaching": {
          "allowResultReuse": true,
          "hit": false,
          "result": "Cache Miss",
          "hashes": {
            "output count": "C4CA4238A0B923820DCC509A6F75849B",
            "runtime attribute": {
              "docker": "N/A",
              "continueOnReturnCode": "CFCD208495D565EF66E7DFF9F98764DA",
              "failOnStderr": "68934A3E9455FA72420237EB05902327"
            },
            "output expression": {
              "File count_file": "EF1B47FFA9990E8D058D177073939DF7"
            },
            "input count": "C4CA4238A0B923820DCC509A6F75849B",
            "backend name": "509820290D57F333403F490DDE7316F4",
            "command template": "FD00A1B0AB6A0C97B0737C83F179DDE7",
            "input": {
              "File input_file": "d3410ade53df34c78488544285cf743c"
            }
          },
         "effectiveCallCachingMode": "ReadAndWriteCache"
        },
        "inputs": {
          "input_file": "f"
        },
        "returnCode": 0,
        "jobId": "34495",
        "backend": "Local",
        "end": "2016-02-04T13:47:56.000-05:00",
        "stderr": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/stderr",
        "attempt": 1,
        "executionEvents": [],
        "start": "2016-02-04T13:47:56.000-05:00"
      }
    ],
    "sc_test.do_gather": [
      {
        "executionStatus": "Done",
        "stdout": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_gather/stdout",
        "backendStatus": "Done",
        "shardIndex": -1,
        "outputs": {
          "sum": 12
        },
        "runtimeAttributes": {
           "failOnStderr": "true",
           "continueOnReturnCode": "0"
        },
        "callCaching": {
          "allowResultReuse": true,
          "hit": false,
          "result": "Cache Miss",
          "hashes": {
            "output count": "C4CA4238A0B923820DCC509A6F75849B",
            "runtime attribute": {
              "docker": "N/A",
              "continueOnReturnCode": "CFCD208495D565EF66E7DFF9F98764DA",
              "failOnStderr": "68934A3E9455FA72420237EB05902327"
            },
            "output expression": {
              "File count_file": "EF1B47FFA9990E8D058D177073939DF7"
            },
            "input count": "C4CA4238A0B923820DCC509A6F75849B",
            "backend name": "509820290D57F333403F490DDE7316F4",
            "command template": "FD00A1B0AB6A0C97B0737C83F179DDE7",
            "input": {
              "File input_file": "e0ef752ab4824939d7947f6012b7c141"
            }
          },
          "effectiveCallCachingMode": "ReadAndWriteCache"
        },
        "inputs": {
          "input_files": [
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt",
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
          ]
        },
        "returnCode": 0,
        "jobId": "34494",
        "backend": "Local",
        "end": "2016-02-04T13:47:57.000-05:00",
        "stderr": "/home/jdoe/cromwell/cromwell-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_gather/stderr",
        "attempt": 1,
        "executionEvents": [],
        "start": "2016-02-04T13:47:56.000-05:00"
      }
    ]
  },
  "outputs": {
    "sc_test.do_gather.sum": 12,
    "sc_test.do_prepare.split_files": [
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_aa",
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_ad"
    ],
    "sc_test.do_scatter.count_file": [
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt",
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
    ]
  },
  "id": "8e592ed8-ebe5-4be0-8dcb-4073a41fe180",
  "inputs": {
    "sc_test.do_prepare.input_file": "/home/jdoe/cromwell/11.txt"
  },
  "labels": {
    "cromwell-workflow-name": "sc_test",
    "cromwell-workflow-id": "cromwell-17633f21-11a9-414f-a95b-2e21431bd67d"
  },
  "submission": "2016-02-04T13:47:55.000-05:00",
  "status": "Succeeded",
  "end": "2016-02-04T13:47:57.000-05:00",
  "start": "2016-02-04T13:47:55.000-05:00"
}
```

cURL:

```
$ curl "http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/metadata?includeKey=inputs&includeKey=outputs"
```

HTTPie:

```
$ http "http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/metadata?includeKey=inputs&includeKey=outputs"
```

Response:
```
HTTP/1.1 200 OK
Server spray-can/1.3.3 is not blacklisted
Server: spray-can/1.3.3
Date: Thu, 01 Oct 2015 22:19:07 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 4286
{
  "calls": {
    "sc_test.do_prepare": [
      {
        "shardIndex": -1,
        "outputs": {
          "split_files": [
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_aa",
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_ad"
          ]
        },
        "inputs": {
          "input_file": "/home/jdoe/cromwell/11.txt"
        },
        "attempt": 1
      }
    ],
    "sc_test.do_scatter": [
      {
        "shardIndex": 0,
        "outputs": {},
        "inputs": {
          "input_file": "f"
        },
        "attempt": 1
      },
      {
        "shardIndex": 0,
        "outputs": {
          "count_file": "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt"
        },
        "inputs": {
          "input_file": "f"
        },
        "attempt": 2
      },
      {
        "shardIndex": 1,
        "outputs": {
          "count_file": "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
        },
        "inputs": {
          "input_file": "f"
        },
        "attempt": 1
      }
    ],
    "sc_test.do_gather": [
      {
        "shardIndex": -1,
        "outputs": {
          "sum": 12
        }
        "inputs": {
          "input_files": [
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt",
            "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
          ]
        },
        "attempt": 1
      }
    ]
  },
  "outputs": {
    "sc_test.do_gather.sum": 12,
    "sc_test.do_prepare.split_files": [
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_aa",
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_prepare/temp_ad"
    ],
    "sc_test.do_scatter.count_file": [
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-0/attempt-2/output.txt",
      "/home/jdoe/cromwell/cromwell-test-executions/sc_test/8e592ed8-ebe5-4be0-8dcb-4073a41fe180/call-do_scatter/shard-1/output.txt"
    ]
  },
  "id": "8e592ed8-ebe5-4be0-8dcb-4073a41fe180",
  "inputs": {
    "sc_test.do_prepare.input_file": "/home/jdoe/cromwell/11.txt"
  }
}
```

The `call` and `workflow` may optionally contain failures shaped like this:
```
"failures": [
  {
    "failure": "The failure message",
    "timestamp": "2016-02-25T10:49:02.066-05:00"
  }
]
```

### Compressing the metadata response

The response from the metadata endpoint can be quite large depending on the workflow. To help with this Cromwell supports gzip encoding the metadata prior to sending it back to the client. In order to enable this, make sure your client is sending the `Accept-Encoding: gzip` header.

For instance, with cURL:
                   
```
$ curl -H "Accept-Encoding: gzip" http://localhost:8000/api/workflows/v1/b3e45584-9450-4e73-9523-fc3ccf749848/metadata
```

## POST /api/workflows/:version/:id/abort

cURL:

```
$ curl -X POST http://localhost:8000/api/workflows/v1/e442e52a-9de1-47f0-8b4f-e6e565008cf1/abort
```

HTTPie:

```
$ http POST http://localhost:8000/api/workflows/v1/e442e52a-9de1-47f0-8b4f-e6e565008cf1/abort
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 241
Content-Type: application/json; charset=UTF-8
Date: Thu, 04 Jun 2015 12:15:33 GMT
Server: spray-can/1.3.3

{
    "id": "e442e52a-9de1-47f0-8b4f-e6e565008cf1",
    "status": "Aborted"
}
```

## GET /api/workflows/:version/backends

This endpoint returns a list of the backends supported by the server as well as the default backend.

cURL:
```
$ curl http://localhost:8000/api/workflows/v1/backends
```

HTTPie:
```
$ http http://localhost:8000/api/workflows/v1/backends
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 379
Content-Type: application/json; charset=UTF-8
Date: Mon, 03 Aug 2015 17:11:28 GMT
Server: spray-can/1.3.3

{
  "supportedBackends": ["JES", "LSF", "Local", "SGE"],
  "defaultBackend": "Local"
}
```

## GET /api/workflows/:version/callcaching/diff

**Disclaimer**: This endpoint depends on hash values being published to the metadata, which only happens as of Cromwell 28.
Workflows run with prior versions of Cromwell cannot be used with this endpoint.
A `404 NotFound` will be returned when trying to use this endpoint if either workflow has been run on a prior version.

This endpoint returns the hash differences between 2 *completed* (successfully or not) calls.
The following query parameters are supported:

| Parameter |                                        Description                                        | Required |
|:---------:|:-----------------------------------------------------------------------------------------:|:--------:|
| workflowA | Workflow ID of the first call                                                             | yes      |
| callA     | Fully qualified name of the first call. **Including workflow name**. (see example below)  | yes      |
| indexA    | Shard index of the first call                                                             | depends  |
| workflowB | Workflow ID of the second call                                                            | yes      |
| callB     | Fully qualified name of the second call. **Including workflow name**. (see example below) | yes      |
| indexB    | Shard index of the second call                                                            | depends  |

About the `indexX` parameters: It is required if the call was in a scatter. Otherwise it should *not* be specified.
If an index parameter is wrongly specified, the call will not be found and the request will result in a 404 response.

cURL:

```
$ curl "http://localhost:8000/api/workflows/v1/callcaching/diff?workflowA=85174842-4a44-4355-a3a9-3a711ce556f1&callA=wf_hello.hello&workflowB=7479f8a8-efa4-46e4-af0d-802addc66e5d&callB=wf_hello.hello"
```

HTTPie:

```
$ http "http://localhost:8000/api/workflows/v1/callcaching/diff?workflowA=85174842-4a44-4355-a3a9-3a711ce556f1&callA=wf_hello.hello&workflowB=7479f8a8-efa4-46e4-af0d-802addc66e5d&callB=wf_hello.hello"
```

Response:
```
HTTP/1.1 200 OK
Content-Length: 1274
Content-Type: application/json; charset=UTF-8
Date: Tue, 06 Jun 2017 16:44:33 GMT
Server: spray-can/1.3.3

{
  "callA": {
    "executionStatus": "Done",
    "workflowId": "85174842-4a44-4355-a3a9-3a711ce556f1",
    "callFqn": "wf_hello.hello",
    "jobIndex": -1,
    "allowResultReuse": true
  },
  "callB": {
    "executionStatus": "Done",
    "workflowId": "7479f8a8-efa4-46e4-af0d-802addc66e5d",
    "callFqn": "wf_hello.hello",
    "jobIndex": -1,
    "allowResultReuse": true
  },
  "hashDifferential": [
    {
      "hashKey": "command template",
      "callA": "4EAADE3CD5D558C5A6CFA4FD101A1486",
      "callB": "3C7A0CA3D7A863A486DBF3F7005D4C95"
    },
    {
      "hashKey": "input count",
      "callA": "C4CA4238A0B923820DCC509A6F75849B",
      "callB": "C81E728D9D4C2F636F067F89CC14862C"
    },
    {
      "hashKey": "input: String addressee",
      "callA": "D4CC65CB9B5F22D8A762532CED87FE8D",
      "callB": "7235E005510D99CB4D5988B21AC97B6D"
    },
    {
      "hashKey": "input: String addressee2",
      "callA": "116C7E36B4AE3EAFD07FA4C536CE092F",
      "callB": null
    }
  ]
}
```

The response is a JSON object with 3 fields:

- `callA` reports information about the first call, including its `allowResultReuse` value that will be used to determine whether or not this call can be cached to.
- `callB` reports information about the second call, including its `allowResultReuse` value that will be used to determine whether or not this call can be cached to.
- `hashDifferential` is an array in which each element represents a difference between the hashes of `callA` and `callB`.

*If this array is empty, `callA` and `callB` have the same hashes*.

Differences can be of 3 kinds:

- `callA` and `callB` both have the same hash key but their values are different.
For instance, in the example above, 

```json
{
  "hashKey": "input: String addressee",
  "callA": "D4CC65CB9B5F22D8A762532CED87FE8D",
  "callB": "7235E005510D99CB4D5988B21AC97B6D"
}
```

indicates that both `callA` and `callB` have a `String` input called `addressee`, but different values were used at runtime, resulting in different MD5 hashes.

- `callA` has a hash key that `callB` doesn't have
For instance, in the example above, 

```json
{
  "hashKey": "input: String addressee2",
  "callA": "116C7E36B4AE3EAFD07FA4C536CE092F",
  "callB": null
}
```

indicates that `callA` has a `String` input called `addressee2` that doesn't exist in `callB`. For that reason the value of the second field is `null`.

- `callB` has a hash key that `callA` doesn't have. This is the same case as above but reversed.

If no cache entry for `callA` or `callB` can be found, the response will be in the following format:

```
HTTP/1.1 404 NotFound
Content-Length: 178
Content-Type: application/json; charset=UTF-8
Date: Tue, 06 Jun 2017 17:02:15 GMT
Server: spray-can/1.3.3

{
  "status": "error",
  "message": "Cannot find call 479f8a8-efa4-46e4-af0d-802addc66e5d:wf_hello.hello:-1"
}
```

If neither `callA` nor `callB` can be found, the response will be in the following format:


```
HTTP/1.1 404 NotFound
Content-Length: 178
Content-Type: application/json; charset=UTF-8
Date: Tue, 06 Jun 2017 17:02:15 GMT
Server: spray-can/1.3.3

{
  "status": "error",
  "message": "Cannot find calls 5174842-4a44-4355-a3a9-3a711ce556f1:wf_hello.hello:-1, 479f8a8-efa4-46e4-af0d-802addc66e5d:wf_hello.hello:-1"
}
```

If the query is malformed and required parameters are missing, the response will be in the following format:

```
HTTP/1.1 400 BadRequest
Content-Length: 178
Content-Type: application/json; charset=UTF-8
Date: Tue, 06 Jun 2017 17:02:15 GMT
Server: spray-can/1.3.3
{
  "status": "fail",
  "message": "Wrong parameters for call cache diff query:\nmissing workflowA query parameter\nmissing callB query parameter",
  "errors": [
    "missing workflowA query parameter",
    "missing callB query parameter"
  ]
}
```

## GET /engine/:version/stats

This endpoint returns some basic statistics on the current state of the engine. At the moment that includes the number of running workflows and the number of active jobs. 

cURL:
```
$ curl http://localhost:8000/engine/v1/stats
```

HTTPie:
```
$ http http://localhost:8000/engine/v1/stats
```

Response:
```
"date": "Sun, 18 Sep 2016 14:38:11 GMT",
"server": "spray-can/1.3.3",
"content-length": "33",
"content-type": "application/json; charset=UTF-8"

{
  "workflows": 3,
  "jobs": 10
}
```

## GET /engine/:version/version

This endpoint returns the version of the Cromwell engine.

cURL:
```
$ curl http://localhost:8000/engine/v1/version
```

HTTPie:
```
$ http http://localhost:8000/engine/v1/version
```

Response:
```
"date": "Sun, 18 Sep 2016 14:38:11 GMT",
"server": "spray-can/1.3.3",
"content-length": "33",
"content-type": "application/json; charset=UTF-8"

{
  "cromwell": 23-8be799a-SNAP
}
```

## GET /engine/:version/status

Cromwell will track the status of underlying systems on which it depends, typically connectivity to the database and to Docker Hub. The `status` endpoint will return the current status of these systems. 

Response:

This endpoint will return an `Internal Server Error` if any systems are marked as failing or `OK` otherwise. The exact response will vary based on what is being monitored but adheres to the following pattern. Each system has a boolean `ok` field and an optional array field named `messages` which will be populated with any known errors if the `ok` status is `false`:

```
{
  "System Name 1": {
    "ok": false,
    "messages": [
      "Unknown status"
    ]
  },
  "System Name 2": {
    "ok": true
  }
}

```
 


## Error handling
Requests that Cromwell can't process return a failure in the form of a JSON response respecting the following JSON schema:

```
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description": "Error response schema",
  "type": "object",
  "properties": {
    "status": {
      "enum": [ "fail", "error"]
    },
    "message": {
      "type": "string"
    },
    "errors": {
      "type": "array",
      "minItems": 1,
      "items": { "type": "string" },
      "uniqueItems": true
    }
  },
  "required": ["status", "message"]
}
```

The `status` field can take two values:
> "fail" means that the request was invalid and/or data validation failed. "fail" status is most likely returned with a 4xx HTTP Status code.
e.g.

```
{
  "status": "fail",
  "message": "Workflow input processing failed.",
  "errors": [
    "Required workflow input 'helloworld.input' not specified."
  ]
}
```

> "error" means that an error occurred while processing the request. "error" status is most likely returned with a 5xx HTTP Status code.
e.g.

```
{
  "status": "error",
  "message": "Connection to the database failed."
}
```

The `message` field contains a short description of the error.

The `errors` field is optional and may contain additional information about why the request failed.
