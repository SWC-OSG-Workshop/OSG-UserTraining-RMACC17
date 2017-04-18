---
layout: lesson
root: ../..
title: Data storage and transfer
---
<div class="objectives" markdown="1">

#### Objectives
*   Discover how to transfer input and output data  
*   Learn the what, why, how, and whens of using Stash
</div>


<h2> Overview </h2>
In this lesson, we will learn the basics of data storage and transfer on OSG Connect. 

<h2> Stash </h2>
Stash is a distributed filesystem for temporary storage on OSG Connect. You can put any input or output files here.

First, log in to OSG Connect:
~~~
ssh username@login.osgconnect.net # Connect to the login node with your username
passwd:       # your password
~~~

Once done, you can change to the `data` directory in your home area:
~~~
$ cd ~/data    # This is your *stash* directory
~~~
{:class="in"}

You can also make data on *stash* publically available on the web. Any data copied or moved to the directory *~/data/public* can be accessed here:

~~~
http://stash.osgconnect.net/~yourusername.
~~~


<h2> File Transfer to Stash </h2> 

We can transfer files from our laptop to *stash* with `scp`, `sftp`, or 
Globus. For small files (<50MB), it is fine to use `scp` or `sftp` utilities. For larger files 
or when your access may be intermittent, we recommend using Globus.
 
To transfer a file called "BigData.tar.gz" via scp from your local laptop to *stash*

~~~
scp BigData.tar.gz username@login.osgconnect.net:~/data/.  #Transfers the file "BigData.tar.gz" using secured copy.
~~~
{:class="in"}

<h2> Downloading data from Stash </h2> 

Let us do an example calculation to understand the use of *stash* and how we download 
the data from the web. We will peform 
molecular dynamics simulation of a small protein in implicit water. To get the
necessary files, we use the *tutorial* command on OSG. 

Log in OSG

~~~
$ ssh username@login.osgconnect.net
~~~

type 

~~~
$ tutorial stash-namd
$ cd ~/osg-stash-namd
~~~

~~~
namd_stash_run.submit #Condor job submission script file.
namd_stash_run.sh #Job execution script file.
ubq_gbis_eq.conf #Input configuration for NAMD.
ubq.pdb #Input pdb file for NAMD.
ubq.psf #Input file for NAMD.
par_all27_prot_lipid.inp #Parameter file for NAMD.
~~~

The file - "par_all27_prot_lipid.inp" is the parameter file and is required for 
the NAMD simulations. The parameter file is common data file for the NAMD
simulations. It is a good practice to keep the common files, like  the parameter file 
in our example, in the *stash* storage.  

~~~
mv par_all27_prot_lipid.inp ~/data/public/.  #moving the parameter file from the local directory ~/osg-namd-stash  to ~/data/public 
~~~

You can view the parameter file appear on WWW

~~~
http://stash.osgconnect.net/+yourusername
~~~

Now we want the parameter file available on the execution (worker) machine when the 
simulation starts to run. As mentioned early, the data on the *stash* is available to 
the execution machines. This means the execution machine can transfer the data from 
*stash* as a part of the job execution. So we have to script this in the job execution 
script. 

You can see that the job execution script "namd_stash_run.sh" has the following lines:

~~~
#!/bin/bash  
source /cvmfs/oasis.opensciencegrid.org/osg/modules/lmod/5.6.2/init/bash #sourcing a shell specific file that adds the module command to your environment
module load namd/2.9  #loading the namd module
wget --no-check-certificate http://stash.osgconnect.net/+username/par_all27_prot_lipid.inp # Getting the data from the *stash*. Insert your username. 
namd2 ubq_gbis_eq.conf  #Executing the NAMD simulation
~~~

In the above script, you will have to insert your "username" in URL address. The
parameter file located on *stash* is downloaded using the #wget# utility.  
 

Now we submit the NAMD job. 

~~~
$ condor_submit namd_stash_run.submit 
~~~

Once the job completes, you will see non-empty "job.out.0" file where 
the standout output from the programs are written as default.   

~~~
$ tail job.out.0

WallClock: 6.084453  CPUTime: 6.084453  Memory: 53.500000 MB
Program finished.
~~~

The above lines indicate the NAMD simulation was successful. 

 
<div class="keypoints" markdown="1">

#### Key Points
* Data on *stash* is quickly accessed by the worker machines. 
* *stash* is located at ~/data on login.osgconnect.net. 
* *scp* and *rsync* are useful tools for data transfer.
</div>

