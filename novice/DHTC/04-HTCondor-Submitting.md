---
layout: lesson
root: ../..
title: Job Scheduling with HTCondor
---
<!-- <div class="objectives" markdown="1">

#### Objectives
*   Learn how to submit HTCondor jobs.
*   Learn how to monitor the running jobs.
</div> -->

## Overview

In this section, we will learn the basics of HTCondor in submitting and monitoring workloads, or "jobs". The typical cycle for a job is:

1. Job is submitted through the submit host
2. Job is executed on the remote worker node.
3. Job executes or fails and the logs and, if configured, outputs are transfered back to the login node. 

In HTCondor we use a job submit file to describe how to execute the program, transfer data, and what the system requirements of a job are.

<!-- ![fig 1](https://raw.githubusercontent.com/OSGConnect/tutorial-quickstart/master/Images/jobSubmit.png) -->


## Job Execution Script

We will get our examples using the `tutorial` command.

Let's get started with the `quickstart` tutorial:

    # creates a directory "tutorial-quickstart"
    $ tutorial quickstart
    # script and input files are inside this directory  
    $ cd tutorial-quickstart

We will look at two files in detail: `short.sh` and `tutorial01.submit`

Inside the tutorial directory, look at `short.sh` using

    $ cat short.sh

This is an ordinary shell script.

    #!/bin/bash
    # short.sh: a short discovery job
    printf "Start time: "; /bin/date
    printf "Job is running on node: "; /bin/hostname
    printf "Job running as user: "; /usr/bin/id
    printf "Job is running in directory: "; /bin/pwd
    echo
    echo "Working hard..."
    sleep 20
    echo "Science complete!"

Now, make the script executable.

    $ chmod +x short.sh

Making the script executable and the "shebang" line (`#!/bin/bash`) line at the top of the script are not necessary for programs that are only run locally. However, _it is extremely important for jobs running on the grid_.

Since we used the `tutorial` command, all files are already in your workspace. Test jobs on the submit host first b locally when setting up a new job type -- it is good to test your job outside of HTCondor before submitting into the Open Science Grid.

    $ ./short.sh
    Start time: Mon Mar  6 00:08:06 CST 2017
    Job is running on node: training.osgconnect.net
    Job running as user: uid=46628(username) gid=46628(username) groups=46628(username),400(condor),401(ciconnect-staff),1000(users)
    Job is running in directory: /tmp/Test/tutorial-quickstart
    Working hard...
    Science complete!

## Job Description File

Next we will create a simple (if verbose) HTCondor submit file.  A submit file tells the HTCondor _how_ to run your workload, with what properties and arguments, and, optionally, how to return output to the submit host.

    $ cat tutorial01.submit

    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla

    # EXECUTABLE is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    Executable = short.sh

    # ERROR and OUTPUT are the error and output channels from your job
    # that HTCondor returns from the remote host.
    Error = job.error
    Output = job.output

    # The LOG file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    Log = job.log

    # QUEUE is the "start button" - it launches any jobs that have been
    # specified thus far.
    queue 1

## Job Submission

Submit the job using `condor_submit`.


    $ condor_submit tutorial01.submit
    Submitting job(s).
    1 job(s) submitted to cluster 1144.


## Job Status

The `condor_q` command tells the status of currently running jobs. Please note that the `condor_q` command line interface has changed in recent HTCondor versions, and in this tutorial we are using the new version.

    $ condor_q

    -- Schedd: training.osgconnect.net : <192.170.227.119:9618?... @ 06/30/17 15:17:51
    OWNER    BATCH_NAME       SUBMITTED   DONE   RUN    IDLE  TOTAL JOB_IDS
    username CMD: short.sh   6/30 15:17      _      _      1      1 1872.0

    Total for query: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended 
    Total for username: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended 
    Total for all users: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended

This new output format "batches" similar jobs together. If you want to see each individual job, use the `-nobatch` option:

    $ condor_q -nobatch

    -- Schedd: training.osgconnect.net : <192.170.227.119:9618?... @ 06/30/17 15:19:21
     ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
    1873.0   rynge           6/30 15:19   0+00:00:00 I  0    0.0 short.sh

    Total for query: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended 
    Total for rynge: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended 
    Total for all users: 1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended

If you want to see all jobs running on the system, use `condor_q -allusers`.

You can also get status on a specific job cluster:

    $ condor_q -nobatch 1144.0 

    -- Schedd: training.osgconnect.net : <192.170.227.119:9419?...
     ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD               
    1144.0   username       3/6  00:17   0+00:00:00 I  0   0.0  short.sh

    1 jobs; 0 completed, 0 removed, 1 idle, 0 running, 0 held, 0 suspended

Note the ST (state) column. Your job will be in the `I` state (idle) if it hasn't started yet. If it's running, it will have state `R` (running). If it has completed already, it will not appear in `condor_q`.

Let's wait for your job to finish – that is, for `condor_q` not to show the job in its output. A useful tool for this is `watch`. It runs a program repeatedly, letting you see how the output changes at fixed time intervals. Let's submit the job again, and watch condor_q output at two-second intervals:

    $ condor_submit tutorial01.submit
    Submitting job(s). 
    1 job(s) submitted to cluster 1145
    $ watch -n2 condor_q $USER 


When your job has completed, it will disappear from the list.  To close watch, hold down Ctrl and press C.

## Job History

Once your job has finished, you can get information about its execution
from the `condor_history` command:

    $ condor_history 1144
    ID     OWNER          SUBMITTED   RUN_TIME     ST COMPLETED   CMD            
    1144.0   osguser50       3/6  00:17   0+00:00:27 C   3/6  00:28 /share/training/..

You can see much more information about your job's final status using the `-long` option.

## Job Output

Once your job has finished, you can look at the files that HTCondor has returned to the working directory. If everything was successful, it should have returned:

* `job.output`: An output file for each job's output
* `job.error`: An error file for each job's errors
* `job.log`: A log  file for each job's log

Read the output file. It should be something like this:


    $ cat job.output
    Start time: Wed Mar  8 18:16:04 CST 2017
    Job is running on node: cmswn2300.fnal.gov
    Job running as user: uid=12740(osg) gid=9652(osg) groups=9652(osg)
    Job is running in directory: /storage/local/data1/condor/execute/dir_2031614/glide_6B4s2O/execute/dir_887949
    Working hard...
    Science complete!


## Removing Jobs

You may want to remove individual or all your workloads from the queue. The command to remove jobs from the queue is `condor_rm`. It takes one argument, either the job ID or your username. Providing the job ID will remove only that job:

    $ condor_submit tutorial01.submit
    Submitting job(s).
    1 job(s) submitted to cluster 1145 
    $ condor_rm 1145
    Cluster 1145 has been marked for removal.

while providing your username will remove all jobs associated with your username:

    $ condor_rm username
    All jobs of user "username" have been marked for removal

## Basics of HTCondor Matchmaking

As you have seen in the previous lesson, HTCondor is a batch management system that handles running jobs on a cluster. Like other full-featured batch systems, HTCondor provides a job queuing mechanism, scheduling policy, priority scheme, resource monitoring, and resource management. Users submit their jobs to a HTCondor scheduler. HTCondor places the jobs into a queue, chooses when and where to run the jobs based upon a policy, carefully monitors the their progress, and ultimately informs the user upon completion or failure. This lesson will go over some of the specifics of how HTCondor selects where to run a particular job.  

HTCondor selects nodes on which to run particular jobs using a matchmaking process.  When a job is submitted to HTCondor, HTCondor generates a set of attributes that the job needs in order to run. These attributes function like classified ads in the newspaper and are called "classad"s. The classads for a job indicate what it is looking for, just like a help wanted ad. For example:

    Requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" && HAS_MODULES == True

Let's examine what a machine classad looks like. This is a two step process, first we get a name for one of the machines, and then we ask `condor_status` to give us the details for that machine (`-long`).

    $ condor_status -pool osg-flock.grid.iu.edu -af Name | head -n 1
    [resource name]
    $ condor_status -long -pool osg-flock.grid.iu.edu [resource name] | sort
    [...]
    HAS_FILE_usr_lib64_libgfortran_so_3 = true
    HAS_MODULES = false
    OSGVO_OS_STRING = "RHEL 6"
    [...]


HTCondor takes a list of classads from jobs and from compute nodes and then tries to make the classads with job requirements with the classads with compute node capabilities. When the two match, HTCondor will run the job on the compute node.

## A Basic OSG Job

You can make use any of these attributes to limit where your jobs go. The `osg-template-job.submit` file is an example of a fairly complete OSG job which you can use as a template submit script when getting started on OSG. Note the `Requirements` and `requests_*` lines.


    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla

    # These are good base requirements for your jobs on OSG. It is specific on OS and
    # OS version, core count and memory, and wants to use the software modules. 
    Requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" && HAS_MODULES == True
    request_cpus = 1
    request_memory = 1 GB

    # EXECUTABLE is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    Executable = short.sh

    # ERROR and OUTPUT are the error and output channels from your job
    # that HTCondor returns from the remote host. $(Cluster) is the 
    # ID HTCondor assigns to the job and $(Process) is the ID HTCondor
    # assigns within a set of jobs.
    Error = job.$(Cluster).$(Process).error
    Output = job.$(Cluster).$(Process).output

    # The LOG file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    Log = job.log

    # Send the job to Held state on failure. 
    on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)  

    # Periodically retry the jobs every 60 seconds, up to a maximum of 5 retries.
    periodic_release =  (NumJobStarts < 5) && ((CurrentTime - EnteredCurrentStatus) > 60)

    # QUEUE is the "start button" - it launches any jobs that have been
    # specified thus far.
    queue 1


You can test this job by submitting and monitoring it as we have just covered:

    $ condor_submit osg-template-job.submit
    Submitting job(s).
    1 job(s) submitted to cluster 1151


The filenames for this job includes a job id, which means that if you submit more
than one job, they will all have unique outputs.

    $ ls *.output
    job.output
    job.1151.0.output

## A More Advanced OSG Job

For any script or job you want to run, you will usually want to do one or several of the following things: pass input parameters to a script, use input file, and produce an output file. Open the example script `short_with_input_output_transfer.sh` with `cat`:

    $ cat short_with_input_output_transfer.sh

This is a shell script that is similar to the above example. The main difference is that this script takes a text file as a command line argument argument, i.e. $1, and produces an output file that is the copy of the input file, i.e. cat $1 > output.txt.

    #!/bin/bash
    # short.sh: a short discovery job
    printf "Start time: "; /bin/date
    printf "Job is running on node: "; /bin/hostname
    printf "Job running as user: "; /usr/bin/id
    printf "Job is running in directory: "; /bin/pwd
    printf "The command line argument is: "; $1
    printf "Contents of $1 is "; cat $1
    cat $1 > output.txt
    printf "Working hard..."
    ls -l $PWD
    sleep 20
    echo "Science complete!"

With this setup will have to transfer the input file to the remote worker node, pass the input file as a command line argument, and then transfer the output file back. We do all these things in the example submit file `osg-template-job-input-and-transfer.submit`:

    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla
    
    # These are good base requirements for your jobs on OSG. It is specific on OS and
    # OS version, core count and memory, and wants to use the software modules. 
    Requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" && HAS_MODULES == True
    request_cpus = 1
    request_memory = 1 GB
    
    # EXECUTABLE is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    Executable = short_with_input.sh
    
    # ERROR and OUTPUT are the error and output channels from your job
    # that HTCondor returns from the remote host. $(Cluster) is the 
    # ID HTCondor assigns to the job and $(Process) is the ID HTCondor
    # assigns within a set of jobs. 
    Error = job.$(Cluster).$(Process).error
    Output = job.$(Cluster).$(Process).output
    
    # The LOG file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    Log = job.log
    
    # Send the job to Held state on failure. 
    on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)  
    
    # Periodically retry the jobs every 60 seconds, up to a maximum of 5 retries.
    periodic_release =  (NumJobStarts < 5) && ((CurrentTime - EnteredCurrentStatus) > 60)

    # TRANSFER_INPUT_FILES defines which files should be transferred to the job. 
    # Please note that this should only be used for relatively small files
    transfer_input_files = input.txt

    # TRANSFER_OUTPUT_FILES defines which files should be transferred from the job back to 
    # the submit host. 
    # Please note that this should only be used for relatively small files
    transfer_output_files = output.txt
    
    # ARGUMENTS is a way to pass command line input to the EXECUTABLE
    arguments = input.txt

    # QUEUE is the "start button" - it launches any jobs that have been
    # specified thus far.
    queue 1

You can test this job by submitting and monitoring it as we have just covered:

    $ condor_submit osg-template-job-input-and-transfer.submit
    Submitting job(s).
    1 job(s) submitted to cluster 1152

The filenames for this job includes a job id, which means that if you submit more
than one job, they will all have unique outputs.

    $ ls *.output
    job.output
    job.1151.0.output
    job.1152.0.output

There will also be an `output.txt` in the directory:

    $ ls output.txt
    output.txt

## Challenges

* What happens if we leave the `queue` line out of a submit file?

* What happens if we write only `queue`, with no argument?


* `condor_history -long username` gives a LOT of extended information about your past jobs, ordered as key-value pairs.  Try it with your a single job from your last cluster:

    $ condor_history -long ######.0


    Included among these attributes is the `RemoteWallClockTime` parameter, which tells how long your job ran on the remote worker.  How might you
collect this value across all your historical jobs? (Remember that the `grep` command can be used to pick out specific patterns from text.)

<!-- <div class="keypoints" markdown="1">

#### Key Points
*   HTCondor shedules and monitors your Jobs.
*   To submit a job to HTCondor, you must prepare the job execution and job submission scripts.
*   *condor_submit* - HTCondor's job submission command.
*   *condor_q* - HTCondor's job monitoring command.
*   *condor_rm* - HTCondor's job removal command.
</div> -->

