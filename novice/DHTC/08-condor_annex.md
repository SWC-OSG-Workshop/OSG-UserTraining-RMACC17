---
layout: lesson
root: ../..
title: Adding Resources from Amazon AWS
---
<div class="objectives" markdown="1">

#### Objectives
*   Discover how to use `condor_annex` to add dedicated resources from Amazon AWS
*   Submit workloads to AWS, and to OSG and AWS simultaneously
*   Please note that his is a technical preview. We expect features and interface to change rapidly.
</div>


<h2>condor_annex (and connect_annex)</h2>

* The `condor_annex` tool is available in the HTCondor 8.7.2 development release
* Labeled as “experimental” because the interface(s) might change
* The current version has limited functionality and hence limited applicability

Use cases

* Deadlines
* Capability - large memory, GPU, long run times, fast local storage, job policies
* Capacity

`connect_annex` is a wrapper around `condor_annex` which provides some OSG Connect
integration. Please use `connect_annex` in this tutorial.


<h2>Setting up AWS access</h2>

    $ mkdir ~/.condor
    $ cd ~/.condor
    $ touch publicKeyFile privateKeyFile
    $ chmod 600 publicKeyFile privateKeyFile

The last command ensures that only you can read or write to those files.

To donwload a new pair of security tokens for `condor_annex` to use, go
to the [IAM console](https://console.aws.amazon.com/iam/home?region=us-east-1#/home)
; log in if you need to. The following instructions
assume you are logged in as a user with the privilege to create new
users. (The 'root' user for any account has this privilege; other
accounts may as well.)

 1. Click the __Add User__ button.
 2. Enter name in the User name box; __annex-user__ is a fine choice.
 3. Click the check box labelled __Programmatic access__.
 4. Click the button labelled __Next: Permissions__.
 5. Select __Attach existing policies directly__.
 6. Type __AdministratorAccess__ in the box labelled "Filter".
 7. Click the check box on the single line that will appear below (labelled __AdministratorAccess__)

![IAM](Images/IAM-NewUserRole.png)

 8. Click the __Next: review__ button (you may need to scroll down).
 9. Click the __Create user__ button.
 10. From the line labelled annex-user, copy the value in the column labelled __Access key ID__ to `accessKeyFile`.
 11. On the line labelled annex-user", click the __Show__ link in the column labelled __Secret access key__; copy the revealed value to `secretKeyFile`.
 12. Hit the __Close__ button.

The 'annex-user' now has full privileges to your account. We're working
on creating a CloudFormation template that will create a user with only
the privileges `condor_annex` actually needs.

<h2>Running the Setup Command</h2>

The following command will setup your AWS account. It will create a
number of persistent components, none of which will cost you anything
to keep around. These components can take quite some time to create;
condor_annex checks each for completion every ten seconds and prints an
additional dot (past the first three) when it does so, to let you know
that everything's still working.

    $ connect_annex -setup
    Creating configuration bucket (this takes less than a minute)....... complete.
    Creating Lambda functions (this takes about a minute)........ complete.
    Creating instance profile (this takes about two minutes)................... complete.
    Creating security group (this takes less than a minute)..... complete.
    Setup successful.

<h2>Checking the Setup</h2>

You can verify at this point (or any later time) that the setup procedure completed successfully by running the following command.

    $ connect_annex -check-setup
    Checking for configuration bucket... OK.
    Checking for Lambda functions... OK.
    Checking for instance profile... OK.
    Checking for security group... OK.
    Your setup looks OK.





<h2>Running jobs on the EC2 instances</h2>

Run the quickstart tutorial:

	$ tutorial annex
	$ cd tutorial-annex

Tutorial jobs
-------------

Inside the tutorial directory you will find a sample executable:

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


### HTCondor submit file

So far, so good! Let's look at a the submit file `ec2-job.submit`

    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla
    
    # These are good base requirements for your jobs on OSG. It is specific on OS and
    # OS version, core cound and memory, and wants to use the software modules. 
    # This is the default recommended OSG requirements:
    #Requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" &&  HAS_MODULES == True
    # To make sure we are only running on EC2, use regex matching on the Machine attribute
    Requirements = regexp("ec2.internal", Machine) 
    request_cpus = 1
    request_memory = 1 GB
    
    # EXECUTABLE is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    Executable = short.sh
    Arguments = 
    
    # ERROR and OUTPUT are the error and output channels from your job
    # that HTCondor returns from the remote host.
    Error = job.$(Cluster).$(Process).error
    Output = job.$(Cluster).$(Process).output
    
    # The LOG file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    Log = job.log
    
    # Send the job to Held state on failure. 
    on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)
    
    # Periodically retry the jobs every 1 hour, up to a maximum of 5 retries.
    periodic_release =  (NumJobStarts < 5) && ((CurrentTime - EnteredCurrentStatus) > 60*60)
    
    # QUEUE is the "start button" - it launches any jobs that have been
    # specified thus far.
    Queue 1


### Submit the job 

Submit the job using `condor_submit`:

	$ condor_submit ec2-job.submit
	Submitting job(s). 
	1 job(s) submitted to cluster 823.

### Check the job status

The `condor_q` command tells the status of currently running jobs.
Generally you will want to limit it to your own jobs: 

	$ condor_q netid
	-- Submitter: login01.osgconnect.net : <128.135.158.173:43606> : login01.osgconnect.net
	 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
	 823.0   netid           8/21 09:46   0+00:00:06 R  0   0.0  short.sh
	1 jobs; 0 completed, 0 removed, 0 idle, 1 running, 0 held, 0 suspended

You can also get status on a specific job cluster: 

	$ condor_q 823
	-- Submitter: login01.osgconnect.net : <128.135.158.173:43606> : login01.osgconnect.net
	 ID      OWNER            SUBMITTED     RUN_TIME ST PRI SIZE CMD
	 823.0   netid           8/21 09:46   0+00:00:10 R  0   0.0  short.sh
	1 jobs; 0 completed, 0 removed, 0 idle, 1 running, 0 held, 0 suspended

Note the `ST` (state) column. Your job will be in the I state (idle) if
it hasn't started yet. If it's currently scheduled and running, it will
have state `R` (running). If it has completed already, it will not appear
in `condor_q`. 


### Check the job output

Once your job has finished, you can look at the files that HTCondor has
returned to the working directory. If everything was successful, it
should have returned:

* a log file from HTCondor for the job cluster: jog.log
* an output file for each job's output: job.output
* an error file for each job's errors: job.error

Read the output file. It should be something like this: 

	$ cat job.output
	Start time: Wed Aug 21 09:46:38 CDT 2013
	Job is running on node: ip-12.12.12.12
	Job running as user: uid=58704(osg) gid=58704(osg) groups=58704(osg)
	Job is running in directory: /var/lib/condor/execute/dir_2120
	Sleeping for 10 seconds...
	Et voila!

The `ip-NNN.NNN.NNN.NNN` indicates that the job ran in the EC2 instance.


### Jobs across OSG and Amazon EC2

We will now edit the `ec2-job.submit` file, so that the requirements 
expression allows the job to go to either OSG or EC2. The new requirements
line should read:

    Requirements = regexp("ec2.internal", Machine) || (OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64")

Let's also run a set of jobs. Change the queue line to read:

    Queue 20

The submit the job again.


### Where did jobs run? 

When we start submitting many simultaneous jobs into the queue, it might
be worth looking at where they run. To get that information, we'll use the
`condor_history` command from quickstart tutorial.Change the job id (942)
to the job id provided by the `condor_submit` command:

	[netid@login01 log]$ condor_history -format '%s\n' LastRemoteHost 942 | cut -d@ -f2 | cut -d. -f2,3 | distribution --height=100
	Val          |Ct (Pct)     Histogram
	ec2.internal |456 (46.77%) +++++++++++++++++++++++++++++++++++++++++++++++++++++
	uchicago.edu |422 (43.28%) +++++++++++++++++++++++++++++++++++++++++++++++++
	local        |28 (2.87%)   ++++
	t2.ucsd      |23 (2.36%)   +++
	phys.uconn   |12 (1.23%)   ++
	tusker.hcc   |10 (1.03%)   ++
	...

The distribution program reduces a list of hostnames to a set of
hostnames with no duplication (much like `sort | uniq -c`), but
additionally plots a distribution histogram on your terminal
window. This is nice for seeing how Condor selected your execution
endpoints.

<div class="keypoints" markdown="1">

#### Key Points
* OSG can schedule jobs to resources brought by the user

</div>

