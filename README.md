# Getting Started with MSI

This documentation is designed to assist you in getting started with MSI by providing guidance on accessing the system, submitting jobs, and adhering to best practices. You are encouraged to supplement this information with additional useful information (or removal of outdated information) as needed in the future.

## Table of Contents

1. [When to use MSI?](#when-to-use-MSI)
2. [Connecting to MSI and file transfer](#connecting-to-MSI)
3. [What are modules on MSI?](#modules)
4. [Running jobs](#running-jobs) \
    4.1 [Interactive Jobs](#interactive-jobs) \
    4.2 [Batch Jobs](#batch-jobs) \
    4.3 [Job Arrays](#job-arrays)
5. [Parallelizing in Julia`](#parallelizing-in-julia)
6. [Installing softwares on MSI](#installing-software-on-MSI) \
    6.1 [Julia](#julia) \
    6.2 [CPLEX](#cplex) \
    6.3 [BARON](#baron) \
    6.4 [Gurobi](#gurobi) \
    6.5 [HSL Package (Linear solvers for IPOPT)](#hsl)
7. [Good practices for different solvers](#good-practices-for-different-solvers)
8. [Miscellaneous troubleshooting tips](#miscellaneous-troubleshooting-tips)
9. [Parallel computing with Julia](#parallel-computing-with-julia)
10. [Common Julia syntax](#common-julia-syntax)
11. [Common errors, work-arounds, and best practices](#common-errors)


<a name="when-to-use-MSI"></a>
## When to use MSI?
Consider using MSI over your PC in the following cases:

- Solving optimization problems or simulations with large compuation time.
- Running experiments requuring more memory access than available on your PC.
- Running computational experiments with a huge number of instances.
- Running parallel computations that require more cores than available on your PC.

MSI has the follwoing Linux computing clusters available for use:
- Agate
- Mangi
- Mesabi [soon to be discontinued]

Each cluster has nodes with different architecture and more information on that can be found [here](https://www.msi.umn.edu/partitions).

<a name="connecting-to-MSI"></a>
## Connecting to MSI and file transfer

To use MSI on **Windows** it is recommended to download the following software to your PC:

**PuTTY** - an SSH client that will allow you to open the MSI terminal - download [here](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

In **Linux and Mac** based computers, do `ssh X500@partition-name.msi.umn.edu`. Replace `partition-name` with the partition you want to access (e.g. `agate`, `mangi`, `mesabi`).

Although you can use the terminal to move files to and from MSI, it is recommended to used a more user-friendly software like WinSCP (for Windows) or FileZilla (for Linux and Mac). More information on these can be found [here](https://www.msi.umn.edu/support/faq/how-do-i-transfer-data-between-mac-or-windows-computer-and-msi-unix-computers).

If you are not on the University of Minnesota network, you will need to use the University's VPN to access MSI. More information on that can be found [here](https://it.umn.edu/services-technologies/virtual-private-network-vpn).

<a name="modules"></a>
## What are modules on MSI?
Modules are softwares that are either installed on the MSI globally (available to all users) or can be locally installed by the user on their own profile. List of softwares available on MSI can be found [here](https://www.msi.umn.edu/software). This list is not always upto date and if you are unsure check with the MSI staff or use `module avail software-name` command to see if the software is available. For example, Gurobi is pre-installed on MSI and can be loaded using the following command:
```
module load gurobi/10.0.1
```
The above command loads Gurobi version 10.0.1. To see all the available versions of Gurobi available on MSI, you can use the following command:
```
module avail gurobi
```
To install newer versions of modules available on MSI, you need to contact the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk). Also, for globally installed modules that require a license (e.g. Gurobi), you need to contact the MSI staff to grant you the access.

The locally installed softwares needs to be placed in a particular folder and the user needs to add that folder to their PATH variable. For example, if you have installed a software in the folder `~/software`, you can add the following line to your `.bashrc` file or directly to your job script.
```
export PATH=$PATH:~/software
```

More on installing specific softwares on MSI can be found [here](#installing-software-on-MSI).

<a name="running-jobs"></a>
## Running Jobs 

<a name="interactive-jobs"></a>
### Interactive jobs
Interactive sessions are useful for debugging and testing your code and maybe if you only have to run one particular instance of optimization/simulation problem, but you need to make sure the terminal is active all the time; hence not recommended for the last task (batch jobs are more suitable). To start an interactive session, you can use the following command:

To start an interactive session, you can use the following command:
```
srun -N 1 --ntasks-per-node=4  --mem-per-cpu=1gb -t 1:00:00 -p interactive --pty bash
```

More info on interactive jobs can be found [here](https://www.msi.umn.edu/content/interactive-queue-use-srun).

<a name="batch-jobs"></a>
### Batch jobs

Batch jobs should be used in most cases, esp. if you have to run a large number of computational experiments. To submit a batch job, you need to create a Slurm job script. A job script is a shell script that contains the necessary commands to run your job. A sample job script is shown below:
```
#!/bin/bash -l        
#SBATCH --time=2:00:00
#SBATCH --ntasks=8
#SBATCH --mem=10g
#SBATCH --tmp=10g
#SBATCH --mail-type=FAIL  
#SBATCH --mail-user=x500@umn.edu 
module load gurobi/10.0.1
module load julia/1.9.3
julia main.jl
```

To run the above job script, you can use the following command:
```
sbatch -p amdsmall job.sh
```

Replace `job.sh` with your slurm job script name. The `-p` flag is optional and assignes your job to the specified partition (`amdsmall` in this case). More info on job scheduling using Slurm can be found [here](https://www.msi.umn.edu/content/job-submission-and-scheduling-slurm).

<a name="job-arrays"></a>
### Job arrays
Job arrays are valuable for executing code with various parameters. While a for loop can serve this purpose, it may not always be practical, especially for computationally intensive tasks. This could result in the need to request nodes for extensive periods, which may exceed the node's usage limit. In such scenarios, job arrays prove beneficial as they enable running the same job multiple times with distinct parameters across different nodes, each assigned a unique job ID.

For example, suppsoe you have the following Julia script that solves an optimization problem: 

```
using JuMP, Gurobi

function solve(a)
    m = Model(Gurobi.Optimizer)
    @variable(m, x>=0)
    @variable(m, y>=0)
    @constraint(m, x + y >= a)
    @objective(m, Min, 2x + y)
    optimize!(m)
    sleep(10) # Simulating a long computation
    return objective_value(m)
end

println(solve(ARGS[1]))
```

You can create a job array to run the above script for different values of `a` by using the `$SLURM_ARRAY_TASK_ID` variable. A sample job script for the above example is shown below:
```
#!/bin/bash -l  
#SBATCH --time=2:00:00
#SBATCH --ntasks=8
#SBATCH --mem=10g
#SBATCH --tmp=10g
module load gurobi/10.0.1
module load julia/1.9.3
julia main.jl $SLURM_ARRAY_TASK_ID
```

To run the above job array, you can use the following command:
```
sbatch -p amdsmall --array=1-10 job.sh
```

The above command will create 10 jobs, each with a different value of `a` ranging from 1 to 10.

<a name="parallelizing-in-julia"></a>
## Parallelizing in Julia
This differs from job arrays in a significant way. While job arrays involve running the same job on various nodes with distinct job IDs, here it's a single job with parallel execution of its components (some part of the code). For instance, in decomposition algorithms, this approach can facilitate parallelizing subproblems.

In Julia, we can utilize the `Distributed` package to parallelize our code (or part of it). This section is meant to be a brief overview of coding syntax. Consider a scenario where you have a Julia script that implements a decomposition algorithm, and you aim to parallelize its subproblems. The following code demonstrates how to achieve this:

```
using Distributed

addprocs(10)

#= ...
  some code before solving subproblems
  ... =#

@everywhere function subproblem(n)
    # solves some optimization problem
    return n^2
end

#= ...
  some code after solving subproblems
  ... =#

z = pmap(subproblem, 1:10; retry_delays = 3)
```

The above code will run the `subproblem` in parallel on 10 different processes. Note that, it doesn't have to be the *_subproblems_* in the code, it can be any part of the code that can be parallelized.

> :warning: While it is possible to generate more processes than the number of cores requested, it's generally discouraged. This is due to the processes competing for the same resources, often leading to performance degradation.

Description of different components (functions/macros) used to setup parallel computing:

* `addprocs(n)` - Make available `n` Julia processes/workers. 

* `@everywhere` - Makes a function/module available on all workers. Note that you can have blocks of code that is performed `@everywhere` by using `begin` and `end`.

* `pmap(n->myfunction(n),range)` - Map the value `n` which takes all values in `range` onto `myfunction`. This is a parallel equivalent of running a for loop over `range` and evaluating the function in every iteration of the `for` loop. Any julia modules/`.jl` files/global variables required by the `myfunction` must be made available `@everywhere`.

This is the most basic way to parallelize your code. Parallelization is a very extensive topic and one should refer the [documentation](https://docs.julialang.org/en/v1/manual/parallel-computing) to learn more about it.

<a name="installing-software-on-MSI"></a>
## Installing softwares on MSI

### Julia

Some versions of Julia are already installed on MSI. To see which versions are available, you can use the following command: `module avail julia`. If you want to use one of the already installed version, simply run `module load julia/version-number`. If the version you want is not available, you can either submit a ticket to the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk) to request the version you want, or you can install it yourself as follows: 

From you MSI's terminal, run the following command to download the Julia:
```
wget https://julialang-s3.julialang.org/bin/linux/x64/1.10/julia-1.10.2-linux-x86_64.tar.gz
tar zxvf julia-1.10.2-linux-x86_64.tar.gz
```
> :caution: Make sure to replace the version number with the version you want to install or see the command [here](https://julialang.org/downloads/platform/).

To be able to use julia when submitting a job to MSI, you need to do two more things. First, you need to build a module for using julia. To do so, create a folder in your home directory called `Modules`. Then, create a folder within Modules called `Julia`. Then, within `Julia`, create a file whose name is the version of Julia you downloaded (e.g. 1.10.2). Inside this file, write the following line of code: 

`#%Module`<br>
`prepend-path PATH /path/to/julia/bin`

### CPLEX

> :caution: Please update if you think the following information is outdated.

To install CPLEX to your home folder, first download the installation file from the IBM academic initiative website. You want the version for Linux x86-64. If you fail to log into or download from the IBM website, make sure you close all the add blockers or change another browser. Once this file is downloaded, use WinSCP to move it to your home directory in MSI. Then, open the MSI terminal in PuTTY and run the installation package using the code `./cplex_studio1210.linux-x86-64.bin`. Follow the prompts given to install CPLEX. Remember, to use CPLEX in Julia on MSI, you will also need to add the JuMP and CPLEX packages in Julia. To use CPLEX in Python, you will need to follow the last prompts in CPLEX installation to install it in python and add CPLEX package under your anaconda environment.

### BARON

To install BARON to your home folder, first download the zipped files [here](https://minlp.com/baron-downloads). Then, unzip the files and upload them to your home directory in MSI. Retrieve the BARON license file and place it in this folder. To use BARON in Julia on MSI, you will need to add the JuMP and BARON packages in Julia, and define the following environment variables:

`export BARON_EXEC=/path/to/BARON/executable`<br>
`export PATH=$PATH:/path/to/BARON/license/folder`

### Gurobi
Gurobi is installed globally on MSI. To use Gurobi, you can load the module using the following command:

`module load gurobi/VersionNumber`

To see all the available versions of Gurobi available on MSI, you can use the following command:

`module avail gurobi`

To request newer versions of Gurobi, submit a ticket to the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk).

### HSL Package (Linear solvers for IPOPT)
HSL solver is an alternative linear solver that can be used in IPOPT. It has better efficiency in some NLP systems compared to MUMPS. To use the HSL solver, you will have to first apply for the academic license and the download link of the Coin-HSL Full (Stable) version, visit [here](http://www.hsl.rl.ac.uk/ipopt/). Once you get the file, extract and upload it onto MSI. Find the "libcoinhsl.so" in the "lib" folder and rename it as "libhsl.so". Add the library path into your environment or add the following line into your pbs script":

`export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/YourHslFolderPath/lib`

Finally, specify the linear solver that you want to use in IPOPT by defining the 'linear_solver' as "ma27". There are other solvers you can use in the HSL package, visit [here](http://www.hsl.rl.ac.uk/ipopt/). You can also install HSL solver by recompiling IPOPT, visit [here](https://coin-or.github.io/Ipopt/INSTALL.html) 

## Good practices for different solvers
### BARON 

Ensuring options file is read when using parallel computing: Julia's BARON package writes a file called `options` to the current active directory, and then deletes it when BARON is done running. The problem is, if you are running multiple instances of BARON in parallel, the options file may be closed while another parallel run is trying to read it. The easiest way to fix this is to define a temporary folder for each different parallel instance you are running, and to change the active directory to that temporary folder at the beginning of the parallel code.

BARON has been observed to run significantly slower on the Mangi queue than on a PC or on Mesabi. The current working theory on this is that something is using Intel's MKL library which Intel has intentionally made incompatible with AMD processors (used by Mangi). Adding the following line to your PBS script may help: `export MKL_DEBUG_CPU_TYPE=5 `.

BARON on MSI is usually not able to locate CPLEX installation by default and tends to use the open-source solver clp for linear problems. Therefore, if you have CPLEX installed, make sure to specify the installation directory using the appropriate option so that BARON can find it.

### CPLEX
<span style="color:red">
[Might be outdated!]
</span>

Node file management: CPLEX writes information about different nodes in the branch-and-bound tree to what are called node files. By default, CPLEX writes these to memory. However, memory is a precious commodity on MSI nodes, so it is better to write these node files to disc (and more specifically, to scratch space). To do so, add the following options to CPLEX when defining the model in Julia: `CPX_PARAM_NODEFILEIND=3` and `CPX_PARAM_WORKDIR=/scratch.local/myx500`.

### Gurobi
Please update if you encounter any issues with Gurobi.

### IPOPT
Please update if you encounter any issues with IPOPT.

## Miscellaneous troubleshooting tips

**ERROR message:** 

ERROR: LoadError: LoadError: IOError: could not spawn `/panfs/roc/groups/10/qizh/yourfile/baron-lin64/baron /tmp/jl_N8EEfQ/baron_problem.bar`: permission denied (EACCES)

**Solution:**
1. Check the right on the specific file: ls -l ~/baron-lin64/baron
2. Make the file to be executable: chmod +x  ~/baron-lin64/baron
3. Double check the right on the file. It should be -rwx------


**ERROR message:** 

```
OpenBLAS blas_thread_init: pthread_create failed for thread 1 of 8: Resource temporarily unavailable
OpenBLAS blas_thread_init: RLIMIT_NPROC 16384 current, 16384 max
```
This implies that the number of processes requested exceeds the max limit. Check the number of processes requested using `addprocs`. Also, removing the `JULIA_NUM_THREADS` command from job script would generally resolve the issue esp. if using `pmap` for parallelization as of version 1.5.0. **__Caution__**: This can have different implications depending on julia version.

<a name="common-errors"></a>
### Common Errors, Work-arounds, and Best Practices

* The dreaded `EOFError`, `Worker xx terminated`, or `ProcessExitedException` - This error can have many causes which are almost never apparent by looking at the stacktrace. In general, it typically means that at least one processor crashed in the running of parallelized code. Sometimes, this is as simple as a bug in that code - try running the function not parallelized first to ensure this is not the case. More commonly, this occurs because the job has run out of memory. Work arounds for this include defining as few things `@everywhere` as possible, and requesting more cores than you actually need for the computation (allowing you to also request additional memory). Using Mangi instead of Mesabi, requesting a full node instead of a partial one, and running all computations on a single node if possible can also help prevent this error. Lastly, adding additional processers using `addprocs` in the Julia code instead of `-p xx` in the PBS script seems to be more stable as of version 1.3.1.

* `Error: On worker x: var not defined`: This happens most often when one of your pmapped functions is trying to access something that you didn't define `@everywhere`. It usually tells you the name of what it can't find so you can easily fix it.

* Defining a random parameter on multiple processors - Never use the syntax `@everywhere var=rand()` - this will perform the rand function separately on every processor and as such, var will be defined differently on each processor. Instead use:<br>
`var=rand()`<br>
`@eval @everywhere var=$var`<br>
which performs the random generation once on processor 1, and then defines var everywhere as its value on processor 1.

* BARON isn't using the options I gave it - Julia's BARON package writes a file called `options` to the current active directory, and then deletes it when BARON is done running. The problem is, if you are running multiple instances of BARON in parallel, the options file may be closed while another parallel run is trying to read it. The easiest way to fix this is to define a temporary folder for each different parallel instance you are running, and to change the active directory to that temporary folder at the beginning of the parallel code.

* In general, it is a good idea to ensure that your function runs without error for one instance before trying to pmap it, and it is a good idea to ensure your script runs correctly with a small number of processors on your PC, if possible, before trying to submit to MSI.