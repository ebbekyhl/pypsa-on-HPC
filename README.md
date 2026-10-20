# **PyPSA-Eur on HPC at ICL**

This is a short guide on how to install and run PyPSA-Eur on the HPC at ICL.
For general information on how to use the HPC, read the instructions [here](https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/getting-started/).

**How the HPC works, in one paragraph:** you connect to a shared *login node*, where you edit files, install software and submit jobs. The actual computation runs on *compute nodes*, which you don't use directly: you write a job script saying what to run and what resources you need, and the scheduler runs it when resources are free.

You need to be on the college network or connected to [Unified Access](https://www.imperial.ac.uk/admin-services/ict/self-service/connect-communicate/remote-access/) to reach the HPC.

---

## **1. Connect to the HPC**

### Windows

#### **For your convenience - Use WinSCP as file explorer:**

If you use Windows, you can install WinSCP and use this to explore your files located on the HPC.
Here, you can also easily transfer data from your local computer to the HPC - but be careful with large files.
For large files (more than a few GBs), you need to submit a job to the batch queue.
* Download WinSCP [here](https://winscp.net/eng/index.php)
* Login with "login.cx3.hpc.imperial.ac.uk" as hostname, then your username and password as your imperial login

Open the prompt on your local computer:
```
ssh username@login.cx3.hpc.imperial.ac.uk
```
and sign in with your Imperial username and password.

Now you should be connected to the HPC!

### macOS

macOS already has SSH built in, so there's nothing to install. Instead of typing the full address each time, define an SSH config file with a short nickname.

* Create the config file (in the Terminal app, on your Mac):
```
mkdir -p ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/config && chmod 600 ~/.ssh/config
open -e ~/.ssh/config
```
* Add the following, replacing `username` with your Imperial username, and save:
```
Host cx3
    HostName login.cx3.hpc.imperial.ac.uk
    User <username>
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 4h
```
The `Control...` lines reuse one login for 4 hours. SSH keys are disabled on the login nodes, so without them you (and VS Code) would be asked for your password on every new connection.

* Connect:
```
ssh cx3
```
* The first time, SSH asks whether you trust the host. Check that the fingerprint matches the ED25519 fingerprint listed on the [RCS SSH page](https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/getting-started/using-ssh/) (`SHA256:Mddqkum9DwRsvrxoJckFP/aIKuN1+41qIKpU2vuZjIg`), then type `yes` and enter your password.

Now you should be connected to the HPC!

### **Recommended (all platforms) - work in VS Code:**

* Install the extension "Remote - SSH" in VS Code.
* Click the blue `><` button in the bottom-left corner and choose "Connect to Host...". On a Mac, choose `cx3` (from your SSH config). On Windows, enter `username@login.cx3.hpc.imperial.ac.uk`.
* A new VS Code window opens, connected to the HPC (the bottom-left corner shows `SSH: ...`). Its terminal, file explorer and editor all work on the HPC. Use "File > Open Folder" to open your project folder.

---

## **2. Download PyPSA-Eur**
* Create a folder for your projects:

```
[user@login ~]$ mkdir -p ~/projects
[user@login ~]$ cd ~/projects
```

* Clone the repository from GitHub (optionally check out a specific release, e.g. the one you use locally):
```
[user@login projects]$ git clone https://github.com/PyPSA/pypsa-eur.git
[user@login projects]$ cd pypsa-eur
[user@login pypsa-eur]$ git checkout v2026.09.0
```

---

## **3. Install the PyPSA-Eur environment**

Recent versions of PyPSA-Eur manage their environment with [pixi](https://pixi.sh). The `envs/linux-64.lock.yaml` file used by older versions of this guide no longer exists.

### Option A (recommended): pixi

* Install pixi into your home directory (only once):
```
[user@login ~]$ curl -fsSL https://pixi.sh/install.sh | bash
```
* Open a new terminal so it picks up pixi, then install the environment (this takes a while):
```
[user@login ~]$ cd ~/projects/pypsa-eur
[user@login pypsa-eur]$ pixi install
```
* Check that it works:
```
[user@login pypsa-eur]$ pixi run snakemake --version
```
Run commands inside the environment by prefixing them with `pixi run`, or enter it with `pixi shell` (and leave it with `exit`).

**Pixi cache warning:** pixi will warn that its cache is on a network filesystem and redirect it to your ephemeral space. This is harmless. To set it permanently and silence the warning:
```
[user@login ~]$ echo 'export PIXI_CACHE_DIR=$EPHEMERAL/pixi-cache' >> ~/.bashrc
[user@login ~]$ source ~/.bashrc
```
(Ephemeral files are deleted after 30 days. That only affects the cache; pixi simply re-downloads packages when needed.)

### Option B: conda

You will need to set up conda, which is done by loading the miniforge module. It is already installed on the HPC but needs to be copied to your home directory. NB: You only need to do the first two lines once.
```
[user@login ~]$ module load miniforge/3
[user@login ~]$ miniforge-setup
[user@login ~]$ eval "$(~/miniforge3/bin/conda shell.bash hook)"
```
Then create and activate the environment from the pinned file for Linux:
```
[user@login ~]$ cd ~/projects/pypsa-eur
[user@login pypsa-eur]$ conda create -n pypsa-eur --file envs/default_linux-64.pin.txt
[user@login pypsa-eur]$ conda activate pypsa-eur
```

---

## **4. Choose solver**
* HiGHS is the open-source choice. It is installed together with PyPSA-Eur, and is enough for small test models.
* For large models, Gurobi might be necessary instead of HiGHS to obtain convergence. The Gurobi Python package is already included in the PyPSA-Eur environment; you only need a licence (see [Gurobi's instructions](https://support.gurobi.com/hc/en-us/articles/4534161999889-How-do-I-install-Gurobi-Optimizer)).

Create a gurobi.lic file and save it to the /rds/general/user/username/home/ folder. You will need to add a path such that the Gurobi solver uses this licence:
```
export GRB_LICENSE_FILE=/rds/general/user/<username>/home/gurobi.lic
```

---

## **5. Test with a small model first**

PyPSA-Eur comes with small test configurations. `config/test/config.electricity.yaml` models Belgium only, for one week, electricity only, using HiGHS. It runs in well under an hour, so it's a good way to check that everything works before running something large.

First do a *dry run* on the login node. It lists every step Snakemake would run, without running anything, and catches setup problems in seconds:
```
[user@login pypsa-eur]$ pixi run snakemake -call --configfile config/test/config.electricity.yaml -n
```
It should end with a table of jobs (about 70 for this config) and no errors. The `retrieve_*` jobs download input data; in our test (October 2026), these ran fine inside a batch job on the compute nodes.

---

## **6. Submitting a job to the HPC**
* Create a file called `job.sh` in the pypsa-eur folder:
```
#!/bin/bash
#PBS -l walltime=02:00:00
#PBS -l select=1:ncpus=4:mem=16gb
#PBS -N pypsa-toy

cd $PBS_O_WORKDIR

export PATH=$HOME/.pixi/bin:$PATH

# Uncomment if using Gurobi:
# export GRB_LICENSE_FILE=/rds/general/user/username/home/gurobi.lic

echo "TMPDIR: $TMPDIR"

pixi run snakemake -c 4 --configfile config/test/config.electricity.yaml
```
* Notes:
  * The `#PBS` lines request resources: maximum run time (`walltime`), one node with 4 cores and 16 GB memory (`select`), and the job name (`-N`).
  * Jobs start in your home directory; `cd $PBS_O_WORKDIR` moves to the folder you submitted from.
  * If you use conda instead of pixi, replace the `export PATH` and `pixi run` lines with `source ~/miniforge3/etc/profile.d/conda.sh`, `conda activate pypsa-eur`, and plain `snakemake ...`.
  * Make the Snakemake core count (`-c 4`) match the `ncpus` you request.
  * Define the configfile you want to run, in this case the `config_electricity` test example from PyPSA-Eur.
  * To get an email when the job ends or fails, add `#PBS -m ae` and `#PBS -M your.email@imperial.ac.uk`.
* Adjust settings where relevant (e.g. walltime, memory, job name, config file). Smaller requests usually start sooner.
* Submit the job (from the pypsa-eur folder):

```
[user@login pypsa-eur]$ qsub job.sh
```

If you are running a myopic pathway optimisation and you are limited by the available walltime (e.g. 72 hours), you may need to restart the job while ensuring that it continues from where it left off. To do this, you can add the `--rerun-triggers mtime` flag, which ensures that successfully solved years are not run again.

### **Attach manually to a computation node**
Let's say you want to run a Python script or a Jupyter notebook on a computation node in an interactive window (i.e. without submitting a job with a .sh file). You can use the following command (allocating cores, memory and walltime):
```
qsub -I -l select=1:ncpus=4:mem=8gb -l walltime=00:30:00
```

Then, once on the computation node, go to your project folder and use the environment:

Using Pixi:
```
cd ~/projects/pypsa-eur
pixi shell -e dev
```

Using Conda:
```
eval "$(~/miniforge3/bin/conda shell.bash hook)"
conda activate pypsa-eur
```

To run a Jupyter server:
```
jupyter notebook --no-browser --ip=0.0.0.0 --port=8888
```

You can access the Jupyter notebook through an SSH tunnel. Note the compute node's name (shown in the prompt, or with `hostname`):
* Windows: e.g. with [MobaXterm](https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/getting-started/using-ssh/#mobaxterm).
* macOS: in a new local Terminal, run `ssh -L 8888:<node-name>:8888 cx3`, then open the link printed by Jupyter (`http://127.0.0.1:8888/...`) in your browser.

#### **Temporary solver files and heavy I/O**

PyPSA-Eur can have some heavy I/O operations during the solving stage. These are much faster on the compute node's local temporary storage (`$TMPDIR`) than on the network filesystem ([more here](https://icl-rcs-user-guide.readthedocs.io/en/latest/hpc/best-practice/#stage-via-tmpdir)).

Older versions of this guide edited `solve_network.py` to make this happen. **This is not needed in recent versions:** the solver interface (linopy) writes its temporary files to the system temporary directory by default, which in a PBS job is `$TMPDIR`. The `echo "TMPDIR: $TMPDIR"` line in the job script prints that location in the job output so you can check it.

---

## **7. Monitoring jobs, logs and results**

* Show your jobs (`Q` = queued, `R` = running; the job disappears from the list when it ends):
```
[user@login ~]$ qstat -u username
```
* Show a finished job, including its exit status (`Exit_status = 0` means success):
```
[user@login ~]$ qstat -x <job-id>
```
* Delete a job, using its job ID (e.g. `403120.pbs-7`):
```
[user@login ~]$ qdel 403120.pbs-7
```
* Show all jobs running on the HPC:
```
[user@login ~]$ qstat
```

**Logs:**
* The job's output files `<jobname>.o<job-id>` and `<jobname>.e<job-id>` appear in the submission folder *only once the job has finished*. Snakemake's progress goes to the `.e` file; a successful run ends with e.g. `72 of 72 steps (100%) done`.
* While the job is running, follow Snakemake's own log:
```
[user@login pypsa-eur]$ tail -50 $(ls -t .snakemake/log/*.log | head -1)
```
* Logs for each step are in `logs/<run-name>/` (e.g. `logs/test-elec/`).

**Results** are in `results/<run-name>/` (e.g. `results/test-elec/`), including the solved networks in `networks/` and plots from the plotting rules.

---

## **8. Analysing results**

TODO...
e.g. What is best practice, analyse results on HPC, download them to local? Other?
