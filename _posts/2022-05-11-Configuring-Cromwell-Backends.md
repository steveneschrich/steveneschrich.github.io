# Overview
Cromwell works by being a workflow engine, namely it takes the definitions of how to do the work and executes them using existing resources. This is the overall strength of any workflow software - separate the specification of what needs doing from the glue that executes it. I know from my experiences, the intermingling of "glue" and content makes code more complicated and difficult to follow. Not to mention porting the code to another system. Cromwell handles this via backends. There are several options and a configuration file that allows you to customize execution to your environment. I was not at all clear on this topic, so below are the notes and details as I understand them. Note that at the moment, I only use local and HPC (cluster) resources not GCP or other platforms.

# Backend Overview
The cromwell docs have a description of backends (https://cromwell.readthedocs.io/en/stable/backends/Backends/) which includes the computational backend details and the backend fileystems. These are related but have unique configuration details.

It appears that jobs are setup to execute then the backend has to deal with making sure files are where they are needed (cloud, shared filesystem) and then the pipeline/workflows/tasks can be executed as needed.

As I understand it, at least for the sfs and local backends, cromwell executes jobs using a specific strategy. In the sfs/local case, write out an execution bash script and execute it. Specific actions are configured by defining key variables in the config file. Therefore commands such as `submit` and `submit-docker` appear to be configurable as ways in which the "run it" operation occurs. So in our case, Slurm would have a specific way to do a `submit`, therefore we would configure it accordingly. However, the assumption is always a shared file system which means certain limitations are imposed on how executions happen. 

## SFS
The `sfs` backend stands for a shared filesystem, which makes sense. Running jobs locally (or on a HPC) generally involve the use of a common filesystem across everything. This probably simplifies not just getting data to the right place, but also the submit scripts and result code monitoring. In other instances, I assume, it's harder to do job monitoring, etc. It appears that both local and HPC configs generally run as the `sfs`.

## Two types of config variables
Thus there is an abstraction layer for the backend. Two types of variables can be defined/overridden in the config file or the task that control the behavior of the backend:

- Actions
- RuntimeAttributes

Actions are the things above (submit, submit-docker) and runtime attributes are things like cpu, memory, etc. How each are specified, processed and customized is really the point of this post.

# Local Example
A good place to start with using the config file is the LocalExample (https://github.com/broadinstitute/cromwell/blob/develop/cromwell.example.backends/LocalExample.conf). This is not a complete config file as noted in the file but allows one to see the configuration associated with the backend before getting into other config options.

## A Simple First Step
For the backend configuration, there is a block of definitions that start at `backend`. There is a `default = ` option in this block, obviously, to pick the default backend to use. NB: How do you choose a different backend on the fly? Finally, there is a `providers` block which allows you to specify a list of available providers (one of which is presumably the default). Finally, each provider is listed with configuration options. A minimal (but not executable) example would be the following:

```
# Include the standard configuration options, which can be overridden
include required(classpath("application"))

backend {
  default = LocalExecution

  providers {
     LocalExecution {
      # The actor that runs the backend. In this case, it's the Shared File System (SFS) ConfigBackend.
      actor-factory = "cromwell.backend.impl.sfs.config.ConfigBackendLifecycleActorFactory"
     }
  }

}
```

A couple of notes on this bare-bones example. First, we have to include the standard configuration options (first line). Next, you can see the "actor-factory" used to instantiate this backend. Here it's the shared filesystem backend, which is the focus of this effort.


## Second Step: Some functionality
The first step configuration is good except that it doesn't do anything! In fact, it appears that cromwell will crash in this case. If I had to guess, there is at least one thing that has to be defined: how to submit a task/job. The default way to submit a job if you are running local is to run the submit script. Here's how that is configured. 

```
 # Submit string
 submit = "/usr/bin/env bash ${script}"
```
If you are unfamiliar, env bash will start bash in a new environment - bash will interpret the script file. How do you get the script variable in the first place? It is actually defined as part of the glue inside cromwell. A number of *submit variables* are defined in cromwell (see below for details).


# Backend Lower Level Details
Skip this unless you are interested in seeing some of the code.

One of the first questions is from the documentation, 

> backends conforming to the Cromwell backend specification

So is there a backend specification that would provide full disclosure of what options can be used? In the cromwell source code (https://github.com/broadinstitute/cromwell/tree/develop/backend) there is a pointer back to the documentation. It is difficult to understand for me, but it appears there is a "contract" in terms of the types of variables that one defines in order to execute a backed but there are several layers of abstraction.

However, because I am focusing on HPC it is clear that there is a generic engine called a Shared File System (I believe) or sfs backend that executes jobs under the assumption of a shared filesystem (obviously). The code for this appears to be at https://github.com/broadinstitute/cromwell/tree/develop/supportedBackends/sfs/src/main/scala/cromwell/backend/sfs. This code runs tasks under some basic assumptions which are not clear, but appear to include: create a submit script, submit the submit script and wait for responses. Because it is a shared filesystem, it waits by looking for an `rc` file which is the call's result code. 

The "knobs" that can be tuned for the backend, from earlier above, come from defining variables in the config file. One of the reasons for this exploration is to understand what variables can be set for different behaviors. I was able to find https://github.com/broadinstitute/cromwell/blob/develop/supportedBackends/sfs/src/main/scala/cromwell/backend/impl/sfs/config/ConfigConstants.scala which describes the available constants (variables in the config file) that are availabe. This is close to what I'm looking for, although I cannot easily see how these plug into the execution (in the code). I've included this below (as of 5/11/2022). 


```
object ConfigConstants {
  /*
  List of config keys that may be used in the application.conf.
  NOTE: hyphen separated
   */
  val SubmitConfig = "submit"
  val SubmitDockerConfig = "submit-docker"
  val KillConfig = "kill"
  val KillDockerConfig = "kill-docker"
  val CheckAliveConfig = "check-alive"
  val ExitCodeTimeoutConfig = "exit-code-timeout-seconds"
  val RuntimeAttributesConfig = "runtime-attributes"
  val RuntimeAttributesCachingConfig = "runtime-attributes-for-caching"
  val JobIdRegexConfig = "job-id-regex"
  val RunInBackgroundConfig = "run-in-background"

  /*
  Runtime attributes that may be specified within the RuntimeAttributesConfig.
   */
  val DockerRuntimeAttribute = "docker"
  val CpuRuntimeAttribute = "cpu"
  val MemoryRuntimeAttribute = "memory"
  val MemoryMinRuntimeAttribute = "memoryMin"
  val MemoryMaxRuntimeAttribute = "memoryMax"
  // See: MemoryDeclarationValidation
  val MemoryRuntimeAttributePrefix = "memory_"
  val MemoryMinRuntimeAttributePrefix = "memoryMin_"
  val MemoryMaxRuntimeAttributePrefix = "memoryMax_"
  val DiskRuntimeAttribute = "disk"
  val DiskRuntimeAttributePrefix = "disk_"
  /*
  List of task names used internally.
  NOTE: underscore separated
   */
  val RuntimeAttributesTask = "runtime_attributes"
  val SubmitTask = "submit"
  val SubmitDockerTask = "submit_docker"
  val KillTask = "kill"
  val KillDockerTask = "kill_docker"
  val CheckAliveTask = "check_alive"

  /*
  Inputs passed into the submit and kill commands.
  NOTE: underscore separated.
   */
  val JobNameInput = "job_name"
  val CwdInput = "cwd"
  val DockerCwdInput = "docker_cwd"
  val DockerCidInput = "docker_cid"
  val DockerScriptInput = "docker_script"
  val DockerStdoutInput = "docker_out"
  val DockerStderrInput = "docker_err"
  val StdoutInput = "out"
  val StderrInput = "err"
  val ScriptInput = "script"
  val JobIdInput = "job_id"
  val JobShellInput = "job_shell"
}
© 2022 GitHub, Inc.
Terms
Privacy
```


Note that some of the above keys are sfs-specific and some are generic (wom.RuntimeAttributeKeys: https://github.com/broadinstitute/cromwell/blob/develop/wom/src/main/scala/wom/RuntimeAttributes.scala). See below for this generic list. There are some things (like gpuCount) that are not present in the sfs config. Though I don't have time to experiment, it would not be surprising if these fields are supported by the sfs runtime hints, but are not clearly documented as such.

```
object RuntimeAttributesKeys {
  val DockerKey = "docker"
  val MaxRetriesKey = "maxRetries"
  /**
    * Equivalent to CPUMinKey
    */
  val CpuKey = "cpu"
  val CpuPlatformKey = "cpuPlatform"
  val CpuMinKey = "cpuMin"
  val CpuMaxKey = "cpuMax"
  /**
    * Equivalent to GPUMinKey
    */
  val GpuKey = "gpuCount"
  val GpuMinKey = "gpuCountMin"
  val GpuMaxKey = "gpuCountMax"
  val GpuTypeKey = "gpuType"
  val DnaNexusInputDirMinKey = "dnaNexusInputDirMin"
  /**
    * Equivalent to MemoryMinKey
    */
  val MemoryKey = "memory"
  val MemoryMinKey = "memoryMin"
  val MemoryMaxKey = "memoryMax"
  val TmpDirMinKey = "tmpDirMin"
  val TmpDirMaxKey = "tmpDirMax"
  val OutDirMinKey = "outDirMin"
  val OutDirMaxKey = "outDirMax"
  val FailOnStderrKey = "failOnStderr"
  val ContinueOnReturnCodeKey = "continueOnReturnCode"
}
```

## Interaction of Actions and Runtime Attributes
One question that prompted this post is how to define runtime attributes within a task. It seems simple, but not well documented. To be concrete about it, if a task defines "memory=4GB" how does this get to the actual job submission? From what I can gather, since I can define the slurm submit script then I would connect the "memory=4GB" from a task to the slurm option for memory. 

### Experimenting with Runtime Parameters
My question was that while I might use the term "memory" I could just as easily use "RAM=4GB". I ran the following experiment. For a task (biowdl multiqc), I added a new runtime parameter `RAM="4GB"`. When running cromwell I got the following error:
```
[2022-05-11 09:48:12,13] [^[[38;5;220mwarn^[[0m] slurm [^[[38;5;2m4ccb64f7^[[0m]: Key/s [memory, RAM] is/are not supported by backend. Unsupported attributes will not be part of job executions.
```

This makes sense since the config file does not say anything about RAM in the runtime-attributes definition. If I add `String RAM` as a definition in the runtime-attributes (application.conf) then the errors no longer happen. This does appear to be a way to communicate runtime hints to the execution, beyond those that a sfs would expect (see above).





There is a variable (in the config) called runtime-attributes which can be defined. Things like cpu, memory, etc can be defined in here. 

## Backend Configurations
Ignoring the low-level details, the cromwell repo includes a list of example backends that can be tailored for use: https://github.com/broadinstitute/cromwell/tree/develop/cromwell.example.backends. As mentioned, it is likely that these can fit my needs in particular (based on sfs as I am). 