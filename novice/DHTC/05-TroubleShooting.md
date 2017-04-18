---
layout: lesson
root: ../..
title: Troubleshooting failed jobs
---
<div class="objectives" markdown="1">

#### Objectives
*   Learn how to troubleshoot failed jobs.
*   Learn how to periodically retry the failed jobs.
</div>

<h2>Overview </h2> 
We will discuss how to check the job failures and ways to correct the failures.  

<h2> Troubleshooting techniques </h2> 

<h3> Diagnostics with condor_q  </h3> 
The *condor_q* command shows the status of the jobs and it can be used to diagnose why jobs are not 
running. The *condor_q* command with the option "-better-analyze" will return detailed
information about the jobs. Since OSG Connect sends jobs to many places, we also need to specify a pool name with the "-pool" flag

~~~
$ condor_q -better-analyze JOB-ID -pool osg-flock.grid.iu.edu
~~~

The detailed information about a job may help us to identify why a job is not running properly. 

Let us do an example. 

~~~
$ ssh username@login.osgconnect.net   #login 
$ passwd                              

$ tutorial error101
$ cd tutorial-erro101
$ nano error101_job.submit

$condor_submit error101_job.submit 
~~~

Let us check the job status by

~~~
condor_q username
~~~

The submitted job remains in the idle state. The job is failed to go through the queue. Now we check the 
output from *condor_q -better-analyze* that will give us additional detail. 

~~~
$ condor_q -better-analyze JOB-ID -pool osg-flock.grid.iu.edu
 
# Produces a long ouput. 
# The following lines are part of the output regarding the job requirements.  

The Requirements expression for your job reduces to these conditions:

         Slots
Step    Matched  Condition
-----  --------  ---------
[0]           0  Memory >= 51200                 ######## BIG MEMORY, NOT AVAILABLE ###### 
[1]       14727  TARGET.Arch == "X86_64"
[2]       14727  TARGET.OpSys == "LINUX"
[3]       14727  TARGET.Disk >= RequestDisk
[4]       14727  TARGET.HasFileTransfer
~~~


By looking through the conditions listed, it becomes apparent that the job requesting a very 
large memory, no machine available to meet the requirements. This is an error we introduced 
puposefully in the script by including a line *Requirements = (Memory >= 51200)* in the file 
"error101_job.submit".   The job is removed from the queue and re-submited after 
correcting the requirement in the submit file. 

~~~
$ condor_rm JOB-ID #remove the specific job that has problem in matching the resources.
$ nano file.submit # Edit the submit file and insert this line: Requirements = (Memory >= 512)
$ condor_submit file.submit
~~~

or you can edit the resource requirement of a job while it is in the idle state. 

~~~
condor_qedit JOB-ID Requirements 'Requirements = (Memory >= 512)' 
~~~

<h3> On your own </h3>
  1) Use the *connect status* command to get a list of pools (e.g., 'uc3-mgt.mwt2.org') <br/>
  2) Edit error101_job.submit to include "requirements = (IS_RCC_Syracuse == True)" before the 'queue' statement <br/>
  3) Use *condor_q -better-analyze* against each pool. Does it match any slots? If so, where? <br/>

<br/>
<br/>
<h3> condor_ssh_to_job </h3> 
This command allows the user to *ssh* on the compute node where the job is running.  Once the command 
is run, the user will be in the job's working directory and can examine the job's environment and run 
commands. 

~~~
condor_ssh_to_job JOB-ID  
~~~

<h3> Held jobs and condor_release </h3>

The jobs go to the held state for several reasons. One of them is due to the failure to transfer the output
files. If this is the case, you might see the output of the analysis something as follows

~~~
$ condor_q -analyze 372993.0
-- Submitter: login01.osgconnect.net : <192.170.227.195:56174> : login01.osgconnect.net
---
372993.000:  Request is held.
Hold reason: Error from glidein_9371@compute-6-28.tier2: STARTER at 10.3.11.39 failed to send file(s) to <192.170.227.195:40485>: error reading from /wntmp/condor/compute-6-28/execute/dir_9368/glide_J6I1HT/execute/dir_16393/outputfile: (errno 2) No such file or directory; SHADOW failed to receive file(s) from <192.84.86.100:50805>
~~~

The important parts are the message indicating that HTCondor couldn't transfer the job 
back (SHADOW failed to receive file(s)) and the part one before the last part is the name of the 
file or directory that HTCondor couldn't find.  This failure is probably due to your application 
encountering an error while executing and then exiting before writing it's output files.  If you think 
that the error is a transient one and won't reoccur, you can run 

~~~
condor_release JOB-ID 
~~~
to requeue the job.  Alternatively, you can use *condor_ssh_to_job* to examine the job environment and investigate further.


<h3> Retries with periodic_release </h3>

We do computing on a hetrogenous environment. It is possible that the jobs are pre-empted due to a partcular
resource limitation (For example, the machine wants to perform someother task or reboot while your job 
is still running).  In such cases, we want to re-submit failed jobs without manual intervention. Condor 
supports the automatic deduction and retries of computing jobs in two steps.
In the first step, a failed job is forced to be on the held state.   Next, the held job is 
periodically re-submitted. These two steps are listed in the following condor submit file: 

~~~
# Send the job to Held state on failure. 
on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)  

# Periodically retry the jobs for 3 times with an interval 1 hour.   
periodic_release =  (NumJobStarts < 3) && ((CurrentTime - EnteredCurrentStatus) > (60*60))
~~~


<div class="keypoints" markdown="1">
#### Keypoints
*    *condor_q -better-analyze JOB-ID* command is useful to diagnose failed jobs. 
*    Failed jobs are automatically deducted and periodically retried  with *on_exit_old* and *periodic_release* in the condor submit files.
</div>


