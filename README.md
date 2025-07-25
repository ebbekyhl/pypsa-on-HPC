# **PyPSA-Eur on HPC at ICL**
By Ebbe Kyhl Goetske, Research Associate @ ICL

This is a short guide on how to install and run PyPSA-Eur on the HPC at ICL.
For general instructions on how to use the HPC, see https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/getting-started/.

#### **For your convenience - Use WinSCP as file explorer:**

If you use Windows, you can install WinSCP and use this to explore your files located on the HPC.
Here, you can also easily transfer data from your local computer to the HPC - but be careful with large files.
For large files (more than a few GBs), you need to submit a job to the batch queue.
* Download WinSCP from https://winscp.net/eng/index.php
* Login with "login.cx3.hpc.imperial.ac.uk" as hostname, then your username and password as your imperial login

#### **Connect to HPC:**
Open the prompt on your local computer.
* In prompt: ssh username@login.cx3.hpc.imperial.ac.uk
* In prompt: username and password

Now you should be connected to the HPC!

#### **Loading conda:**

You will need to setup conda which is done by loading the miniforge module
which is already installed on the HPC but needs to be copied to your user
home directory. NB: You only need to do this step once.
* First, run the following lines on the login node:
```
[user@login ~]$ module load miniforge/3
[user@login ~]$ miniforge-setup
```
* Then, load conda:
```
[user@login ~]$ eval "$(~/miniforge3/bin/conda shell.bash hook)"
```
* Now, you are ready to create your new work environment in conda!

#### **Download PyPSA-Eur:**
* Create folder for your projects:

```
[user@login ~]$ cd home
[user@login ~]$ mkdir projects
[user@login ~]$ cd projects
```

* Clone repository from GitHub:
```
[user@login projects]$ git clone https://github.com/PyPSA/pypsa-eur.git
```

* Install PyPSA-Eur environment:
```
[user@login projects]$ cd pypsa-eur
[user@login pypsa-eur]$ conda env create -f envs/linux-64.lock.yaml
```

* Activate PyPSA-Eur environment:
```
[user@login pypsa-eur]$ conda activate pypsa-eur
```

#### **For your convenience - Text editor in VS Code:**

* Work on your code in a text editor - I recommend VS Code
* From VS Code, you can access the HPC directly: Hit the blue bottom-left button to "Open a Remote Window"
* Choose "SSH", then sign in as described above

#### **Choose solver:**
* Highs is the open-source choice. It is installed together with PyPSA-Eur.
* For large models, gurobi might be necessary to use instead of Highs to obtain convergence.
* To install gurobi, follow the instructions given here: https://github.com/martavp/pypsa-in-genomedk

#### **Important feature to avoid overloading the HPC with heavy I/O operations:**

PyPSA-Eur can have some heavy I/O operations during the solving stage which can slow things down. 
This can be overcome by ensuring that temporary files are saved to the temporary storage, which is much faster.
First, change the following configuration in the config file to:
```
use_shadow_directory: true
```
  
Second, change this in the config file:

```
solver_dir: $TMPDIR
```
See more here: https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/best-practice/#stage-via-tmpdir

#### **Submitting your first job to the HPC:**
* First create a bash file:
```
#!/bin/bash
#PBS -l walltime=01:00:00
#PBS -l select=1:ncpus=2:mem=4gb
#PBS -N pypsa-eur

cd $PBS_O_WORKDIR

source ~/miniforge3/etc/profile.d/conda.sh

conda activate pypsa-eur

snakemake -call all --configfile config/test/config.overnight.yaml -j 10
```

* Save this file as "job.sh"
* Then submit the job as:

```
[user@login pypsa-eur]$ qsub job.sh
```

#### **Logs**
Debugging your code?
* all logs are found in the folder "logs/<jobname>" or in the hidden folder "./snakemake/log/"
  
#### **Useful commands:**

* Show all jobs running on the HPC:
```
[user@login ~]$ qstat
```

* Show the jobs run by "user":
```
[user@login ~]$ qstat -u user: Shows the jobs run by "user"
```

* Delete jobs:
```
[user@login ~]$ qdel 403120.pbs-7
```
