# Cromwell
I use cromwell (running wdl workflows) regularly as part of my work. It can be a bit challenging at times, but it does a good job of trying to hide some of the (massive) complexity associated with running complex genomics workflows.

My latest attempt, based on the excellent biowdl project, is to create a "data project" style. Basically using cromwell as a series of complex one-off programs. See https://github.com/steveneschrich/cromwell-proj for details.

## Containers
One thing that I have worked on significantly (and was inspired by biowdl) was that *everything* in cromwell is containerized. We use apptainer, which has worked well for encapsulated the runtime environment of various tools using publicly available docker images for these tools.

## Reference files
Typically in a WDL workflow, one might need to include a genome reference file or other reference file(s). While the default behavior of cromwell can be to make hard links to `File` objects in WDL, one can override this with symlinks. Which works well generally when running on the same system (or in the cromwell terms, in the shared filesystem driver). However, for me it was always a challenge to think aboug passing in 4 files representing a group of "reference" files. 

One solution is to pass in zip files or the like, but these would have to be extracted and subsequently waste space. 

My solution ended up combining containers with bind mounts. Briefly, I can specify a bind mount (source and target) as a string (e.g., `"/somewhere/reference_files:/ref"`) for a task. When the task runs, this bind mount means the container will attach the files in `/somewhere/reference_files` as `/ref` in the container.  I modified the runtime attributes of the cromwell backend (configuration file) to attach the bind mounts that are listed as variables in the task's runtime attributes, so that this process works well for me. 

Now, I refer to a directory containing all the necessary reference files and it is mounted and accessible to the programs I run. Fantastic, I saved space and typing by using this approach. But, of course, there is always a downside.

## Chicken and Egg
I use as many biowdl tasks as possible in my work, to avoid reimplementing wdl code which can be very time consuming. I was working on an RNASeq project using the STAR aligner. In biowdl (and per recommendations from the STAR developer), the amount of memory to allocate (on a HPC or cloud) is related to the genome index, not to the individual files to align. That makes sense, since each read can be aligned so that memory is managed efficiently outside of the genome to align to.

This is where my problems came in. It turns out, WDL has a great way to calculate the sizes of the files dynamically, in WDL. So in biowdl, 
```wdl
Int memoryGb = 1 + ceil(size(indexFiles, "GiB") * 1.3)
```
works very well to size the task. *However*, using my wonderful scheme of bind mounting the index files to the container, the files are not available to cromwell prior to startup of the task. At which point you cannot ask for memory allocation for the task (at least within a HPC environment). Therefore, by the time the WDL code is being evaluated to determine the total RAM required, the RAM must have already been allocated. Hence the problem.

Note my solution was simply to set a largish number within a config file to override the calculated value. But clearly this solution will need some further work.