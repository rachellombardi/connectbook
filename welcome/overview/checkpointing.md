Submtiting Checkpointing Jobs
Checkpointing is a technique that provides fault tolerance for a user’s application. It consists of saving snapshots of a job’s progress so the job can be restarted without losing its progress. We encourage checkpointing as a solution for jobs that will exceed the 72 hour max default runtime in the HTC. 

This section is about writing jobs for an executable which periodically saves checkpoint information, and how to make HTCondor store that information safely, in case it’s needed to continue the job on another machine or at a later time.
This section is not about how to checkpoint a given executable; that’s up to you or your software provider.
Requirements
Your self-checkpointing code may not meet all of the following requirements. In many cases, however, you will be able to add a wrapper script, or modify an existing one, to meet these requirements. (Thus, your executable may be a script, rather than the code that’s writing the checkpoint.) If you can not, consult Working Around the Assumptions and/or the Other Options.
Your executable exits after taking a checkpoint with an exit code it does not otherwise use.
If your executable does not exit when it takes a checkpoint, HTCondor will not transfer its checkpoint. If your executable exits normally when it takes a checkpoint, HTCondor will not be able to tell the difference between taking a checkpoint and actually finishing; that is, if the checkpoint code and the terminal exit code are the same, your job will never finish.
When restarted, your executable determines on its own if a checkpoint is available, and if so, uses it.
If your job does not look for a checkpoint each time it starts up, it will start from scratch each time; HTCondor does not run a different command line when restarting a job which has taken a checkpoint.
Starting your executable up from a checkpoint is relatively quick.
If starting your executable up from a checkpoint is relatively slow, your job may not run efficiently enough to be useful, depending on the frequency of checkpoints and interruptions.


There are two types of checkpointing: exit-driven and eviction-driven. 

In many cases, exit driven checkpointing is preferred over eviction driven checkpointing. Exit driven checkpointing tells a job to end after a user-specified amount of time and to save the job with an exit code value of 85. Upon hitting this time limit, HTCondor will save all files and transfer them to a temporary directory. HTCondor then knows to requeue this job and to fetch the previously saved files prior to beginning an analysis. This approach requires that the user specifies in their code to resume from a checkpointed file. 

Discuss exit driven vs eviction driven
Submit file changes

```
# exit-driven-example.sub

checkpoint_exit_code        = 85
transfer_output_files       = example.checkpoint
should_transfer_files       = yes
when_to_transfer_output     = ON_EXIT

executable                  = exit-driven.sh
# arguments                   = 

output                      = example.out
error                       = example.err
log                         = example.log

cpu                         = 1
request_disk                = 2
request_memory              = 1


queue 1
```


``` exit-driven.sh

#!/bin/bash

# The `timeout` command will stop the job after 4 hours (4h) or whatever number of hours or minutes you set. Insert the execution and arguments that you want to complete in the job by replacing ` do_science argument1 argument2…`.

timeout 4h do_science argument1 argument2…

# Uses the bash notation of `$?` to call the exit value of the last executed command and to save it in a variable called `timeout_exit_status`. 

timeout_exit_status=$?

# Programs typically have an exit code of `124` while they are actively running. This code replaces exit code `124` with code `85`. HTCondor recognizes code `85` and knows to end a job with this code but to then to resume it. If an exit code of `124` is not observed, for example if the program is done running, HTCondor will end the job and have it exit the queue or will place the job on hold if it encounters an error. 

if [ $timeout_exit_status -eq 124 ]; then
    exit 85
fii

exit $timeout_exit_status
```

Process of checkpointing:  
Upon exiting the job, HTCondor will save the existing files to a directory called /spooled. Your job will remain in the queue until it matches again to another slot, where the files (including the most recent checkpointed file) from the /spooled directory will be transferred back to an execute machine where your job will resume.
 
 
timeout 4h do_RAxML_science arg1 arg2...
 
 

 
The line: 
 
timeout_exit_status=$?
 
Uses the bash notation of `$?` to call the exit value of the last executed command and to save it in a variable called `timeout_exit_status`. Timeout will exit with code `124` if the command reaches the time limit and is timed out. If the command completes before the specified time limit, it will return the exit status of the command (and therefore will exit HTCondor normally). 
 
By using this code chunk: 
 
if [ $timeout_exit_status -eq 124 ]; then
  exit 85
fi
 
exit $timeout_exit_status
 
We are saying that if an exit code of 124 is observed (meaning RAxML is still running), change that value to be exit code 85. Exit code 85 notifies HTCondor to end the job and then to resume it. Upon exiting the job, HTCondor will save the existing files to a directory called /spooled. Your job will remain in the queue until it matches again to another slot, where the files (including the most recent checkpointed file) from the /spooled directory will be transferred back to an execute machine where your job will resume.
 


# CHECKPOINTING
# If run is interrupted, simply rerun with same command line and input data with checkpoint ".cps.gz" file from incomplete run in directory.
 
timeout 15m ./iqtree/bin/iqtree -s AGNifAlign107.fasta -st AA -pre treetest -nt AUTO -ntmax 8 -m MFP -msub nuclear -mrate G -bb 1000 -alrt 1000

timeout_exit_status=$?

if [ $timeout_exit_status -eq 124 ]
then
        exit 85
fi
 
exit $timeout_exit_status

Timeout will exit with code `124` if the command reaches the specified time limit and is timed out. HTCondor knows to resume jobs that exit with this code. If the command completes before the specified time limit because the job has already completed, it will return with an ex
# iqtree-test.sub
# Submission file for IQ-Tree tree search job
#
# Specify the HTCondor Universe (vanilla is the default and is used
#  for almost all jobs) and your desired name of the HTCondor log file,
#  which is where HTCondor will describe what steps it takes to run 
#  your job. Wherever you see $(Cluster), HTCondor will insert the 
#  queue number assigned to this set of jobs at the time of submission.
universe = vanilla
log = $(SUBMIT_FILE)_$(Cluster).log
#
# Specify your executable (single binary or a script that runs several
#  commands), arguments, and a files for HTCondor to store standard
#  output (or "screen output").
#  $(Process) will be a integer number for each job, starting with "0"
#  and increasing for the relevant number of jobs.

executable = iqtree-test.sh
# arguments = 
output = $(SUBMIT_FILE)_$(Cluster).out
error = $(SUBMIT_FILE)_$(Cluster).err
#
# Specify that HTCondor should transfer files to and from the
#  computer where each job runs. The last of these lines should be
#  used if there are any other files needed for the executable to use.
checkpoint_exit_code = 85
should_transfer_files = YES
when_to_transfer_output = ON_EXIT
transfer_input_files = AGNifAlign107.fasta, ../software/iqtree
#
# Tell HTCondor what amount of compute resources
#  each job will need on the computer where it runs.
request_cpus = 8
request_memory = 1GB
request_disk = 50MB
#
# Run one instance of job
Queue


#!/bin/bash
#
# iqtree-test.sh
# IQ-Tree automatic model selection and tree search
#
# Run IQ-Tree automatic model selection and tree search. Specify path to iqtree executable from sta
ging directory on compute node.
# -s <filename>: alignment filename
# -st AA: specify amino acid data type
# -pre <prefix>: Specify a prefix for all output files
# -nt AUTO: automatically optimize number of cores
# -ntmax <#>: specify the maximal number of cores to automatically allocate (based on number of cor
es requested in scheduler script)
# -m MFP: Perform extended model selection immediately followed by tree inference using the best-fit model
# -msub nuclear: restrict to nuclear models
# -mrate G: restrict to gamma rate models
# -bb 1000: specify 1000 ultrafast bootstrap replicates
# -alrt 1000: specify 1000 replicates to perform SH-aLRT
#
#
# CHECKPOINTING
# If run is interrupted, simply rerun with same command line and input data with checkpoint ".cps.gz" file from incomplete run in directory.
 
timeout 15m ./iqtree/bin/iqtree -s AGNifAlign107.fasta -st AA -pre treetest -nt AUTO -ntmax 8 -m MFP -msub nuclear -mrate G -bb 1000 -alrt 1000

timeout_exit_status=$?

if [ $timeout_exit_status -eq 124 ]
then
        exit 85
fi
 
exit $timeout_exit_status

#
# keep this job running for a few minutes so you'll see it in the queue:
# sleep 180


https://github.com/ChristinaLK/checkpoint-basics

Rachel Lombardi  9:22 AM
More checkpointing stuff: https://opensciencegrid.org/virtual-school-2021/materials/checkpoint/part1-ex1-checkpointing/

 

