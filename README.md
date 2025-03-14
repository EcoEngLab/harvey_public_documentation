# Introduction to Harvey

High-performance computing (HPC) is essential for research requiring
large datasets, complex models, or simulations. Harvey, the PawarLab’s
computing cluster, allows us to run resource-intensive tasks in
parallel, improving efficiency and reducing processing time.

## Cluster Specifications

Harvey offers the following:

-   **Processor:** Single node with two AMD EPYC 9634 CPUs
-   **Threads:** 168 cores and 336 threads
-   **Memory:** 768GB of RAM, enabling efficient handling of
    memory-intensive tasks
-   **Storage:** 6.7TB of combined storage across all users
-   **GPU Support:** We have applied to an academic grant from NVIDIA
    for GPUs to accelerate computing in machine learning and simulation
    tasks

## HPC Basics

Please familiarise yourself with the High-Performance Computing training
material on
[TheMulQuaBio](https://mhasoba.github.io/TheMulQuaBio/intro.html) to
understand the basics of using HPC! **(Coming soon…! Contact George for
these)**

## Accessing and Logging into Harvey

You must be given permission to use Harvey. If you would like access
please contact George Kalogiannis (<g.kalogiannis23@imperial.ac.uk>) or
Prof. Samraat Pawar (<s.pawar@imperial.ac.uk>).

Each user is provided with a unique login to Harvey. The permissions you
have once logging in will depend on your seniority and needs; for
example, undergraduate and master’s students running parallelised
simulations will not be granted sudo privileges unless necessary for
their work.

To access Harvey, you must be logged in on an Imperial College internet
connection, or make use of their [unified access
services](https://www.imperial.ac.uk/admin-services/ict/self-service/connect-communicate/remote-access/unified-access/).
In a terminal window run:

``` bash
ssh <username>@harvey.dept.imperial.ac.uk
```

Similarly, you can transfer files to and from your computer to Harvey
using either `sftp` or `scp`. See below for `sftp`:

``` bash
cd /directory with-or-without files/ # change to the directory you want to files to/from
sftp <username>@harvey.dept.imperial.ac.uk # log in
get /directory/files # get files from harvey into a local directory
put /directory/files # put files from local into a harvey directory
```

## Anaconda and Software

The user directories in Harvey are set up as below:

``` bash
home/
|-- software/
|-- user-1/
|-- user-2/
```

There is a software directory, where large-in-size software with complex
installation methods (e.g. RepeatMasker) are installed, to avoid
multiple installations taking up storage unnecessarily. Users should
make use of this directory when installing software packages that may be
useful to future/current members in the lab. User-specific changes,
e.g. Nextflow configuration files, should be copied from the software
directory to the user’s directory and modified there - **please do not
modify the files in the software directory unless it is necessary for
them to run on Harvey.**

All users have access to anaconda3. They do not need to install this
independently - only activate from their user home directory as such:

``` bash
source ../software/anaconda3/bin/activate
```

## Scheduling Jobs

Harvey uses SLURM to schedule jobs. This is different to the Imperial
College HPC’s that use PBS. To schedule a job on Harvey, you should use
a bash script formatted in the following way:

``` bash
#!/bin/bash
#SBATCH --time=0-00:30:00     # Maximum time limit (30 minutes)
#SBATCH --ntasks=1            # Number of tasks
#SBATCH --cpus-per-task=1     # Number of CPU cores per task
#SBATCH --mem=300G            # Memory per node
#SBATCH --partition=large_336 # Must keep this requirement on anything running on Harvey

**run your code here**
```

For array jobs, include the requirements as such: `--array=1-999`.
Harvey allows for a maximum of 1000 submitted and/or running array jobs
per user.

There are no upper limits in the time you can run your job for. The
`--cpus-per-task` are configured to use the cpu threads, with an upper
limit of 336 on Harvey, and an upper limit of 760GB of RAM.

Keep in mind, you must use directories relative to the location the bash
file is being run from in your code. You can then submit this to the
scheduler using the following command:

``` bash
sbatch <bash-file.sh>
```

Please refer to the [SLURM
Documentation](https://slurm.schedmd.com/pdfs/summary.pdf) if you would
like to include additional job configurations or for alternative methods
of scheduling batch jobs, such as directly from the terminal :

``` bash
sbatch --job-name=<job name> --ntasks=1 --cpus-per-task=1 --mem=300G --time=00:30:00 
       --partition=large_336 --wrap="python myscript.py"
```

## Jupyter Notebooks

Jupyter Notebooks can utilise multiple cores if the SLURM job requests
it, but this depends on how the workload within the notebook is
structured. SLURM will allocate the requested resources (e.g., multiple
CPU cores), but the actual utilisation of those cores within the Jupyter
Notebook depends on the code being executed.

First, make a job script in your directory that starts a notebook at a
selected unique port number. A port number is needed to allow your local
machine to connect to the Jupyter Notebook server running on the remote
machine through SSH tunneling, enabling secure communication between the
two.

An example script can be seen below, using port `8888`, 4 threads, and
jupyter access to our user home directory. There are no necessary
changes to be made to the `jupyer-notebook` code here, unless the port
is unavailable:

``` bash
#!/bin/bash
#SBATCH --job-name=jupyter-notebook # Job name
#SBATCH --output=jupyter_%j.log     # Log file
#SBATCH --time=0-02:00:00           # Maximum run time
#SBATCH --ntasks=1                  # Number of tasks
#SBATCH --cpus-per-task=4           # Number of threads
#SBATCH --mem=4G                    # RAM size
#SBATCH --partition=large_336       # Harvey partition

# Load any necessary modules or activate your Python environment
conda activate myenv    # Activate your personal conda environment if necessary

# Set up a port for Jupyter
PORT="8888"

# Run Jupyter Notebook
jupyter-notebook --no-browser --ip=0.0.0.0 --port=$PORT --notebook-dir=${HOME}
```

To access the notebook, run the following on your local machine:

``` bash
ssh -N -L <port number>:localhost:<port number> <username>@harvey.dept.imperial.ac.uk
```

Then navigate to `http://localhost:<port number>` in your web browser,
if this does not automatically open.

## Harvey Maintenance

Harvey runs at a stable CPU temperature of 35-40<sup>∘</sup>C and
average temperature under sustained load of 60-65<sup>∘</sup>C - these
are normal operating temperatures. In our case, anything approaching
85-90<sup>∘</sup>C indicates significant air constraints and sustained
heavy load. Although not necessarily concerning, this is something that
Samraat or George should be notified about to further investigate.

CPU temperatures can be checked by running the `watch sensors` command
(may require `sudo`), which will display the first 12 cores of each CPU.
Temperatures will be similar across all cores. You can check the work
load that may be causing thermal issues using either `htop` or `squeue`.

**If temperatures surpass 85-90<sup>∘</sup>C, all processes should be
stopped immediately and George and Samraat should be notified.**

Please keep in mind that Harvey’s hardware is incredibly efficient and
redundant by design. Should anything that could possibly damage the
system occur, it will likely shut down by itself. **Better be safe than
sorry** and notify someone if you suspect there are faults with the
system.

Harvey is generally very loud under normal load and becomes even louder
under high load, to sustain the system long term and to prevent hardware
malfunction. If we notice or get told it is excessively loud, please
check the temperatures as above and notify someone if you notice
anything concerning.

## Assistance

If you would like assistance with using Harvey, whether this is for
computing or storage purposes, please contact George Kalogiannis. Try to
stick to good file organisation practices to help me help you - though
success is not guaranteed, regardless!
