# Cromwell/WDL Start to Finish
As part of a project I'm working on, I need to run the MultiQC (https://multiqc.info/) tool on some RNA Sequencing results. Yes, I could create a conda environment, submit a job on our slurm cluster using a simple batch script, and be done.

On the other hand, I could try and run a Cromwell (https://cromwell.readthedocs.io/en/stable/) pipeline using WDL (https://openwdl.org/). It would be a good way to get started using WDL for day-to-day activities and I know there are a number of tools/repos out there to make this easier. So I figured I'd write a simple step-by-step guide so I can remember how I did it.

## WDL Resources
The key to doing this without investing the next year on the project is the work done by a number of groups, two of which I have been studying closely over the last few weeks:

- BioWDL (https://biowdl.github.io/) is a project that has created some excellent pipelines, tasks, and infrastructure for WDL. Note that the github repo is at (https://github.com/biowdl), which I can never seem to find a link from their page to the repos. They appear to have a scheme for WDL documentation, which is important since these workflows can get really messy.

- St. Judes Bioinformatics Workflows (https://stjudecloud.github.io/workflows/). There are a number of workflows here as well. A tool I am particularly interested in is Oliver (https://stjudecloud.github.io/oliver/) which helps in launching jobs that interact with Cromwell in server mode. They also seem to have a scheme for WDL documentation. 

# Setup

## My Environment
I'm using RHEL 8.3 on a slurm cluster.

## Getting WDL
First, you need to download the cromwell tool(s). These are accessible on github https://github.com/broadinstitute/cromwell/releases. Note there is a womtool and a cromwell jar file to download. I put these in my `~/bin` directory. Assuming a recent java environment, that's all that is needed.

## Getting BioWDL
As mentioned above, I'm going to try and use BioWDL rather than recreate the wheel. My thinking is that they have a large library of tasks available (tasks are the units of work that run a program). They do have a MultiQC task available, which is perfect. I used the latest release is 5.0.1, which seems fine for my purposes. You could always clone their repo to get the most up-to-date versions.

```
wget https://github.com/biowdl/tasks/archive/refs/tags/v5.0.1.tar.gz
```

I extracted this into a `biowdl` directory (`biowdl/tasks-5.0.1`).

# Developing a Workflow
The first step after having everything setup is to develop a workflow to run the multiqc. Note: It is possible to run an entire biowdl workflow but it includes many steps, only one of which is multiqc. So instead we will develop our own workflow.

## A Simple Workflow
A workflow is the part of WDL that (presumably) runs multiple tasks together, either in parallel or serial. Therefore, I created a simple workflow for calling MultiQC against my data:

```
version 1.0

import "biowdl/tasks-5.0.1/multiqc.wdl" as multiqc
workflow MultiQCWorkflow {
    input {
        Array[File] reportDirs
        String dockerImage
    }

    call multiqc.MultiQC as multiqcTask {
        input:
            reports = reportDirs,
            dockerImage = dockerImage
    }
}
```

Of course, that's not how it looked the first time I wrote it. Fortunately, the `womtool` can help make sure there are no syntax errors, etc.

```
java -jar ~/bin/womtool-78.jar validate multiqc.wdl
```

## Input Parameters
With a validated workflow in place, the next step is to define the various parameters that the workflow/task needs. We can rely on the `womtool` again for this.

```
java -jar ~/bin/womtool-78.jar inputs multiqc.wdl > inputs.json
```

This creates an `inputs.json` file with all of the parameters listed (in JSON format). Since biowdl provides an optional/required tag for the parameters then we can just focus on the required parameters (for now). The only two parameters needed are:

```
{
  "MultiQCWorkflow.dockerImage": "ewels/multiqc:v1.12",
  "MultiQCWorkflow.reportDirs": ["/share/dept/work/rnaseq_pipeline"]
}
```

The dockerImage is a place to get a docker image (obviously). I found this on dockerhub and appears to be written by the multiqc author. The reportDirs is an array (note the `[]`) but with only one element.

# Running the Workflow
With the workflow defined and the input parameters described, we are ready to run the workflow! Well, there are a couple more steps to go.

## Configuring Cromwell
Cromwell has a nice facility to configure specific run-time setups through a config file. The help pages (https://cromwell.readthedocs.io/en/stable/Configuring/) are good for this. We start by creating a file `application.conf` with the following

```
include required(classpath("application"))

webservice {
  port = 64503
}

system.io {
  number-of-attempts = 5
}

...

```

There are a number of settings here, which I'm not going to review. A couple of important links:
- singularity containers (https://cromwell.readthedocs.io/en/stable/tutorials/Containers/#singularity). Note there is a section describing a config for slurm+singularity, this is what I used.

- slurm config (https://cromwell.readthedocs.io/en/stable/backends/SLURM/)

## Running Cromwell
For simple use, you don't need to use the server mode for cromwell. I do however wrap the call to cromwell in a bash script so that I can store the specific call.

```
#!/bin/sh

#SBATCH --ntasks 1
#SBATCH --time 1-0:00:00
#SBATCH --cpus-per-task 2
#SBATCH --job-name multiqc
#SBATCH --mem 32G

cd /share/dept/work/multiqc
ml load Java

java -Dconfig.file=application.conf -jar ~/bin/cromwell-78.jar run multiqc.wdl --inputs=inputs.json 
```

A quick `squeue --me` shows that this is successfully running. Note that multiqc takes a while to crawl a directory.

## Running Cromwell from the Server

It is possible however to run Cromwell from the server mode. For completeness I decided I would try this with the `oliver` tool from St. Jude's.

An overview of the process looks something like this:
- Launch the cromwell server
- Use the oliver tool to submit a workflow to that server to execute using the server's URL
