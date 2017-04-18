---
layout: lesson
root: ../..
title: condor_annex on OSG Connect
---
<div class="objectives" markdown="1">

#### Objectives
*   Discover how to use condor_annex to bring resources from Amazon AWS
</div>

Please note that his is a technical preview. We expect features and interface
to change rapidly, and if you have feedback, please sent it to
user-support@opensciencegrid.org

<h2>condor_annex</h2>

* The condor_annex tool was first released last week, in HTCondor 8.7.0
* Labeled as “experimental” because the interface(s) might change
* First version has limited functionality and hence limited applicability

Use cases

* Deadlines
* Capability - large memory, GPU, fast local storage, job policies
* Capacity

<h2>Setting up OSG Connect for AWS access</h2>

Once you have created an `annex` user in the AWS IAM web interface, you
will to run aws configure on the HTCondor submit host (training.osgconnect.net).
When prompted, enter your AWS Access Key, AWS Secret Access Key from your saved
credentials.csv file. You will also need to set the default region to us-east-1,
and change the default output format type to json, as shown below:

    [osguser00@training ~]$ aws configure
    AWS Access Key ID [None]: ****************4FSQ
    AWS Secret Access Key [None]: ****************RbV6
    Default region name [None]: us-east-1
    Default output format [None]: json


<h2>setup-annex</h2>

To launch HTCondor workers on AWS, you’ll need to run the setup-annex
script provided on training.osgconnect.net. This script requires the
following three options: 

    --keypair : The name of the keypair installed into the root account of the Annex 
    --vpc     : The Virtual Private Cloud to be used by HTCondor Annex
    --subnets : The subnetwork used in your VPC

For example:

    [osguser00@training ~]$ setup-annex --keypair lb-condor-test --vpc vpc-c4c27ca1 --subnets subnet-a4958b8c
    Using 'hnqcr3juafqfx7pg' as project ID.
    The stack will be named 'htcondor-annex-condor-grid-uchicago-edu-hnqcr3juafqfx7pg'.
    Checking to see if annex already exists... no.
    Checking VPC for suitability:
        DNS resolution... enabled
        DNS hostnames... enabled
    VPC is suitable.
    Creating private S3 bucket to store pool password... done.
    Uploading pool password file... done.
    Uploading config file... done.
    Starting annex (creating stack)... done.
    Waiting for annex to create autoscaling groups... currently 0....................... .......................... done.
    Splitting annex's desired size among 1 autoscaling groups... 1 done.
    Waiting for annex to become size 6... currently 0........... done.
    Waiting for count of annex instances in pool to become 5... done.

For purposes of the demo, the setup-annex helper script will create 1 worker node with a spot market bid of $0.30/hr. Each worker node has 4 cores and 7.5GB of RAM. 


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

