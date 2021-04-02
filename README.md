# Using Reproman with TACC

## Prerequisites

- [install git/git-annex](https://handbook.datalad.org/en/latest/intro/installation.html)
- [install docker](https://docs.docker.com/get-docker/) or [install singularity (linux only)](https://sylabs.io/guides/3.7/admin-guide/admin_quickstart.html#installation-from-source)
- [install python 3 (if it's not already available)](https://www.python.org/downloads/)
- [have a TACC account (I will be covering lonestar5 in particular)](https://portal.tacc.utexas.edu/)
- [add ssh keys](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

To run singularity images on TACC through reproman, you will need to
load the `tacc-singularity` module on lonestar.
Log into lonestar and open `~/.bashrc` in your favorite editor.
Look for section 1, and place `module load tacc-singularity`
in the appropriate spot.
```bash
############
# SECTION 1
#
# There are three independent and safe ways to modify the standard
# module setup. Below are three ways from the simplest to hardest.
#   a) Use "module save"  (see "module help" for details).
#   b) Place module commands in ~/.modules
#   c) Place module commands in this file inside the if block below.
#
# Note that you should only do one of the above.  You do not want
# to override the inherited module environment by having module
# commands outside of the if block[3].

if [ -z "$__BASHRC_SOURCED__" -a "$ENVIRONMENT" != BATCH ]; then
  export __BASHRC_SOURCED__=1

  ##################################################################
  # **** PLACE MODULE COMMANDS HERE and ONLY HERE.              ****
  ##################################################################

  # module load git
  module load tacc-singularity
fi
```



## Step 1: Install Datalad and Reproman

**DO NOT COPY THE DOLLAR SIGN $ (or any text leading up to the $)**

Create a virtual environment to install:
- datalad: version control scripts and data
- reproman: run scripts on remote machines (and/or locally in parallel)
- datalad_container: run scripts using singularity containers

We will call this environment `.reprotut`:
```bash
python3 -m venv .reprotut
$ source .reprotut/bin/activate # linux
$ .reprotut/scripts/Activate.ps1 # windows
```

Next we need to upgrade `wheel` as a temporary fix for installing `reproman`
```bash
(.reprotut) $ pip install --upgrade wheel
```

Now we can install `reproman`, `datalad`, and `datalad_container`
```bash
(.reprotut) $ pip install -r requirements
```

## Step 2: Create a Datalad Dataset

[datalad create docs](http://docs.datalad.org/en/stable/generated/man/datalad-create.html)
```bash
(.reprotut) $ datalad create -c text2git neuroscout-analysis
```

## Step 3: Add a singularity container to neuroscout-analysis

If you installed singularity:
```bash
(.reprotut) $ datalad containers-add \
--dataset ./neuroscout-analysis/ \
neuroscout-cli \
--url docker://neuroscout/neuroscout-cli:version-0.4.2 \
--call-fmt "singularity exec --cleanenv {img} {cmd}"
```

## Step 4: Add lonestar5 as a resource for reproman

[reproman create docs](https://reproman.readthedocs.io/en/latest/generated/man/reproman-create.html)
- `lonestar-runner`: [arbtrary name for resource](https://homestarrunner.com/)
- `-t`: lonestar5 is accessible via ssh
- `-b`: backend parameters to automate how to authenticate with lonestar5, replace with your username
```bash
(.reprotut) $ reproman create lonestar-runner -t ssh -b user=jdk3232 -b host=ls5.tacc.utexas.edu -b key_filename=~/.ssh/id_rsa
```

## Step 4: Run the neuroscout-cli using Reproman on TACC

[reproman run docs](https://reproman.readthedocs.io/en/latest/execute.html#run)
- `--follow`: I do not want access to my terminal until the job is completed
- `-r`: Use the `lonestar-runner` resource we created in the previous step
- `--sub`: lonestar5 uses slurm to batch submit jobs
- `--orc`: we are using datalad locally, but datalad is not installed on lonestar
- `--jp`: job parameters to modify the submission to slurm
  - `num_nodes`: how many machine nodes do I want (each one has about ~48 threads)
  - `launcher`: tacc does not support [job arrays](https://github.com/ReproNim/reproman/issues/557) so [launcher was the solution](https://github.com/TACC/launcher)
  - `queue`: [one of lonestar5 queues](https://portal.tacc.utexas.edu/user-guides/lonestar5#running-queues)
  - `num_processes`: number of processes to run in parallel on a node (`-n` on slurm)
  - `walltime`: estimate of how long the job should take (in minutes, max: 2880 or 48h)
  - `container`: container to copy over to TACC to run the analysis
  - `root_directory`: directory that reproman initializes and runs in
- `-o`: outputs from command
- The remaining parameters are what get passed into the singularity container.
```bash
reproman run \
--follow \
-r lonestar-runner \
--sub slurm \
--orc datalad-local-run \
--jp num_nodes=1 \
--jp launcher=true \
--jp queue=normal \
--jp num_processes=24 \
--jp walltime=500 \
--jp container=neuroscout-cli \
--jp root_directory=/work/07723/jdk3232/lonestar \
-o "ns-output" \
run \
ns-output \
M8Zv5 \
--neurovault disable \
--n-cpus 24 \
--work-dir /scratch/07723/jdk3232/data
```
**KNOWN ISSUE**: If you try to run this exact same command twice, it will fail, you need to either `chmod -R +rw` on the reproman directory made in TACC or remove the directory on TACC, not ideal, but should be fixed: https://github.com/ReproNim/reproman/issues/578 
## Step 5: Reflect

Could you use this workflow?

