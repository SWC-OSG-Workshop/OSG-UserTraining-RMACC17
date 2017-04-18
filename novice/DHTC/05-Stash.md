---
layout: lesson
root: ../..
title: Data storage and transfer
---
<div class="objectives" markdown="1">

#### Objectives
*   Discover how to transfer input and output data
*   Learn about StashCache, stashcp
*   Run a StashCache based Blast workload
</div>


<h2> Overview </h2>

In this lesson, we will learn the basics of data storage and transfer for DHTC jobs and on OSG Connect. 

<h2>Introduction to Data Handling for DHTC jobs</h2>

Data handling is one of the trickiest parts of effectively using DHTC computing.
New users often run into issues in regards to effectively transferring inputs to compute
jobs or with getting output back.  However, most situations can be broken down
into a few basic use cases: 

*   Jobs with small inputs and outputs
*   Jobs with large inputs but small outputs
*   Jobs with small inputs but large outputs
*   Jobs with both large inputs and large outputs

We will consider each use case one by one.

<div class="keypoints" markdown="1">
#### What is Big Data?
In this module we use the terms *big data*, *small* and *large*. These are somewhat relative
terms. What is considered to be big data on your workstation (e.g. difficult to process due 
to the size) is different from what is considered big on on the grid, cloud or super
computers. Data amounts in the context of OSG is most of the time referring to the amount
of data that needs to be transferred to the compute nodes. Large data can be for example
*a single job accessing 50 GBs* or, *100,000 jobs accessing 100 MBs*.
</div>

<h3>Jobs with small inputs or outputs</h3>

Jobs without large inputs or outputs are the easiest case to handle.   If jobs
have inputs and outputs that are smaller than ~1 gigabyte then the builtin HTCondor
file transfer mechanims will work to handle transfers to and from the compute node.  
Input files can be specified using the `transfer_input_files` option in your
submit file.  Likewise output files and directions can be specified using
`transfer_output_files`.  Once the input and output files are given to HTCondor,
it will automatically transfer the input files to compute nodes when jobs are
scheduled to run and then transfer the output after the job completes back to
your submit node.

<h3>Jobs with large inputs but small outputs</h3>

Jobs with large inputs but small outputs are a little harder to handle.
Examples of these types of jobs, would be things such as BLAST which does
searches for similar DNA sequences.  The inputs (DNA reference databases and DNA
sequence of interest) can easily be tens or hundreds gigabytes.  However, the
output (locations of similar sequences and similarity of these sequences) tends
to be kilobytes or megabytes in size.  

If the majority of the inputs are databases or reference files a possible
solution would be to place the input files on a publicly accessible webserver.
Then jobs would wget the appropriate files when they start running.  Since most
OSG sites have a Squid server, this will minimize the network traffic outside of
the site running your jobs.  If the majority of the input is in reference files
that are identical between jobs, then another alternative would be to use a
service such as OASIS that will make the reference files available on the
majority of OSG sites.  Jobs can then access the files as if they are on the
compute node and only the files that are accessed will be transferred.

Further down we will introduce another solution: StachCache and stashcp.

Since the output files are small (i.e. < ~1GB), using the `transfer_output_files` option
in your submit file and allowing HTCondor to manage transferring outputs from
the compute nodes to the submit node should work without problems.

<h3>Jobs with small inputs and large outputs</h3>

This use case deals with jobs that generate large outputs from a small set of
inputs.  For example, a simulation may generate a large data set showing the
evolution of a physical system based on a small set of input parameters.  

The inputs for this type of jobs can be handled using the HTCondor transfer
mechanisms.  By using the `transfer_input_files` option in your submit file,
HTCondor will automatically handle transferring inputs to compute nodes when
your job runs.  

The outputs for this type of job are more difficult to handle.  Solutions may
range from compressing the outputs to generate smaller, more manageable files to
setting up or utilizing a service like gridftp to copy files back to your login
node or a storage system for later retrieval.  Regardless, it is probably best
to talk to us so that we can come up with the best solution for your transfer
needs. 

<h3>Jobs with both large inputs and large outputs</h3>

This use case is essentially just a combination of the prior two use cases.  As
such, inputs can be handled using a web server or OASIS and transferring outputs
will probably need some discussion with us to determine the best solution.

<h3>Guidelines for Data Storage</h3>

|   | **Recommended Limit**| **Purpose** | **Network mounted** | **Backed Up** | **Quota** | **Purge** | **Details**|
|:------- |:----------------:|:------|:------:|:------:|:------:|:------:|:----------|
| **home**    |  < 20 GB      | Meant for quick data access and not for submitting jobs.| Yes | Yes | 20 GB | No | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **local-scratch**   |  < 25 GB      | Meant for large temporary storage and I/O for your jobs. Files older than 30 days are automatically removed. | No | No | No | Yes, 30 days | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **stash**   |  N/A          | Meant for large storage and I/O for your jobs. | Yes | No | No | no | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|
| **public**  |  N/A          | Meant for sharing data and transfer input data via HTTP or staschcp | Yes | No | No | No | [Data Storage](https://support.opensciencegrid.org/support/solutions/articles/12000002985-storage-solutions-on-osg-home-stash-and-public)|

<h2>Exploring the Stash system</h2>

First, we'll look at accessing Stash from the login node.  You'll need to log in to OSG Connect:

~~~
ssh username@training.osgconnect.net #Connect to the login node with your username
passwd:       # your password
~~~

Once done, you can change to the `stash` directory in your home area:

~~~
$ cd ~/stash
~~~

This directory is an area on Stash that you can use to store files and
directories.  It functions just like any other UNIX directory although it has
additional functions that we'll go over shortly.

For future use, let's create a file in the public part of Stash:

~~~
$ cd ~/stash/public/
$ echo "Hello world" > my_hello_world.txt
~~~

In addition, let's create a directory as well for future use:

~~~
$ mkdir my_directory
~~~

One way of accessing this data is over HTTP. Open a web browser and go to the URL:

~~~
http://stash.osgconnect.net/+username/
~~~


<h2>Transferring files to and from Stash using SCP </h2> 

We can transfer files to Stash using `scp`.  `Scp` is a counterpart to ssh that allows for
secure, encrypted file transfers between systems using your ssh credentials.    

To transfer a file from Stash using `scp`, you'll need to run `scp` with the
source and destination.  Files on remote systems are indicated using
user@machine:/path/to/file .  Let's copy the file we just created from Stash to
our local system:

~~~
$ scp username@training.osgconnect.net:~/stash/public/my_hello_world.txt .
~~~

As you can see, `scp` uses similar syntax to the `cp` command that you were shown
previously.  To copy directories using `scp`, you'll just pass the `-r` option to
it.  E.g:

~~~
$ scp -r username@training.osgconnect.net:~/stash/public/my_directory .
~~~

> #### Challenges
>
> * Create a directory with a file called `hello_world_2` in the `~/stash/public` directory and copy it from Stash to your local system.
> * Create a directory called `hello_world_3` on your local system and copy it to the `stash` directory.

<h2>Stash, staschp and stashcache</h2>

StashCache is a data service for OSG which transparently caches data
near compute sites for faster delivery to active grid jobs. The standard
practice of using HTCondor file transfer or http to move data files to
grid sites can be inefficient for certain workflows. StashCache uses a
distributed network filesystem (based on XRootD proxy caching) which
provides an alternative method for transferring input files to compute
sites. It is implemented through a handful of strategically-distributed
sites which provide caching of the input data files.

<h3>When to use StashCache</h3>

StashCache typically outperforms other methods in the following cases:

* Jobs require large data files (> 500 MB)
* The same data files are used repeatedly for many separate jobs

<h3>How to use StashCache</h3>

1)  Include the following line in the job's submit script to indicate that StashCache is required:

    +WantsStashCache = true

2)  Use the 'stashcp' command-line tool in the job wrapper script to transfer necessary data files to the compute host.

First load the stashcp module:

~~~
$ module load stashcp
~~~

Then transfer your data:

~~~
$ cd $HOME
$ stashcp /user/<username>/public/my_hello_world.txt $HOME/
~~~


<h2>StashCache Blast</h2>

This tutorial will use a
[BLAST](http://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastH
ome) workflow to demonstrate the functionality of StashCache for
transferring input files to active job sites. BLAST (Basic Local
Alignment Search Tool) is an open-source bioinformatics tool developed
by the NCBI that searches for alignments between query sequences and a
genetic database.

The input files for a BLAST job include:

* one or more query files
* genetic database file
* database index files

In this exercise, the database is contained in a single 1.3 GB file, and
the query is divided into 2 files of approximately 7.5 MB each (small
enough to use the HTCondor file transfer mechanism). Each query must be
compared to the database file using BLAST, so 2 jobs are needed in this
workflow. If BLAST jobs have an excessively long run time, the query
files can be further subdivided into smaller segments.

Usually, the database files used with BLAST are quite large. Due
to its size (1.3 GB) and the fact that it is used for multiple
jobs, the database file and corresponding index files will
be transferred to the compute sites using StashCache, which
takes advantage of proxy caching to improve transfer speed and
efficiency. Learn more about StashCache and basic usage instructions
[here](https://support.opensciencegrid.org/solution/articles/12000002775
-introduction-to-stashcache).

~~~
$ tutorial stashcache-blast
$ cd tutorial-stashcache-blast
~~~

The tutorial-stashcache-blast directory contains a number of files, described below:

* HTCondor submit script: **blast.submit**
* Job wrapper script: **blast_wrapper.sh**
* Query files: **query_0.fa  query_1.fa**

In addition to these files, the following input files are needed for the jobs:

* database file: **nt.fa**
* database index files: **nt.fa.nhr  nt.fa.nin  nt.fa.nsq**

These files are currently being stored in `/cvmfs/stash.osgstorage.org/user/eharstad/public/blast_database/`.

***
First, let's take a look at the HTCondor job submission script:

    $ cat blast.submit
    universe = vanilla

    executable = blast_wrapper.sh
    arguments  = blastn -db /cvmfs/stash.osgstorage.org/user/eharstad/public/blast_database/nt.fa -query $(queryfile)
    should_transfer_files = YES
    when_to_transfer_output = ON_EXIT
    transfer_input_files = $(queryfile)

    +WantsCvmfsStash = true
    requirements = OSGVO_OS_STRING == "RHEL 6" && Arch == "X86_64" && HAS_MODULES == True && HAS_CVMFS_stash_osgstorage_org == True

    output = job.out.$(Cluster).$(Process)
    error = job.err.$(Cluster).$(Process)
    log = job.log.$(Cluster).$(Process)

    # For each file matching query*.fa, submit a job
    queue queryfile matching query*.fa

The executable for this job is a wrapper script, `blast_wrapper.sh`, that takes as arguments the blast command that we want to run on the compute host.  We specify which query file we want transferred (using HTCondor) to each job site with the *transfer_input_files* command.

Note the one additional line that is required in the submit script of any job that uses StashCache:

    +WantsCvmfsStash = true

Finally, since there are multiple query files, we submit them with the command `queue queryfile matching query*.fa` command.  Because we have used the $(queryfile) macro in the name of the query input files, only one query file will be transferred to each job.

***
Now, let's take a look at the job wrapper script which is the job's executable:

    $ cat blast_wrapper.sh
    #!/bin/bash
    # Load the blast module
    module load blast

    set -e

    "$@"

The wrapper script loads the blast modules so that it can access the Blast software on the compute host.

You are now ready to submit the jobs:

    $ condor_submit blast.submit

 Each job should run for approximately 3-5 minutes.  You can monitor the jobs with the condor_q command:

    $ condor_q <username>


<div class="keypoints" markdown="1">

#### Key Points
* Data can be transferred in and out of Stash using scp and Globus
</div>

