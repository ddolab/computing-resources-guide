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
5. [Parallel computing with Julia](#parallelizing-in-julia)
6. [Common errors and best practices while running parallel computations in Julia](#common-errors)
7. [Installing software on MSI](#installing-software-on-MSI) \
    7.1 [Julia](#julia) \
    7.2 [CPLEX](#cplex) \
    7.3 [BARON](#baron) \
    7.4 [Gurobi](#gurobi) \
    7.5 [HSL Package (Linear solvers for IPOPT)](#hsl)
8. [Good practices/common issues with solvers](#issues-with-different-solvers) \
    8.1 [BARON](#baron-1) \
    8.2 [CPLEX](#cplex-1) \
    8.3 [Gurobi](#gurobi-1) \
    8.4 [IPOPT](#ipopt-1)
9. [Miscellaneous troubleshooting tips](#miscellaneous-troubleshooting-tips)


<a name="when-to-use-MSI"></a>
## When to use MSI?
Consider using MSI over your PC in the following cases:

- Solving optimization problems or simulations with large computation time.
- Running experiments requiring more memory access than is available on your PC.
- Running computational experiments with a huge number of instances.
- Running parallel computations that require more cores than are available on your PC.

MSI has the following Linux computing clusters available for use:
- Agate
- Mangi
- Mesabi [retiring soon]

Nodes on each cluster have different architecture, and more information on that can be found [here](https://www.msi.umn.edu/partitions).

<a name="connecting-to-MSI"></a>
## Connecting to MSI and file transfer

To use MSI on **Windows**, it is recommended to download [PuTTY](https://www.msi.umn.edu/support/faq/how-do-i-configure-putty-connect-msi-unix-systems), which is an ssh client that will allow you to connect to MSI terminal. Instead, you can also use PowerShell.

In **Linux- and Mac**-based systems, run `ssh X500@cluster-name.msi.umn.edu` command in the terminal. Replace `cluster-name` with the cluster you want to access (e.g. `agate`, `mangi`, `mesabi`).

Although you can use the terminal to move files to and from MSI, it is recommended to use softwares like WinSCP (for Windows) and FileZilla (for Linux and Mac) with user-friendly GUIs. More information on these can be found [here](https://www.msi.umn.edu/support/faq/how-do-i-transfer-data-between-mac-or-windows-computer-and-msi-unix-computers).

If you are not on the University of Minnesota network, you will need to use the University's VPN to access MSI. More information on that can be found [here](https://it.umn.edu/services-technologies/virtual-private-network-vpn).

<a name="modules"></a>
## What are modules on MSI?
Modules are softwares that are either installed on the MSI globally (available to all users) or can be locally installed by the user on their own profile. A list of software available on MSI can be found [here](https://www.msi.umn.edu/software). This list is not always up to date, and if you are unsure, check with the MSI staff or use the `module avail software-name` command to see if the software is available. For example, Gurobi is pre-installed on MSI and can be loaded using the following command:
```
module load gurobi/10.0.1
```
The above command loads Gurobi version 10.0.1. To see all versions of a module available on MSI, you can use the following command:
```
module avail module-name
```
To install newer versions of modules available on MSI, you need to contact the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk). Also, for globally installed modules that require a license (e.g., Gurobi), you need to contact the MSI staff to grant you access.

Locally installed softwares needs to be placed in a particular folder, and the user needs to add that folder to their `PATH` variable (using the `export` command) either directly in the `.bashrc` file or in the Slurm job script (see [batch jobs](#batch-jobs)) before loading the software via `module load local-software`.

More on installing specific software on MSI can be found [here](#installing-software-on-MSI).

<a name="running-jobs"></a>
## Running Jobs 

<a name="interactive-jobs"></a>
### Interactive jobs
Interactive sessions are useful for debugging and testing your code. It can also be used in projects that only require running a single instance of an optimization or simulation problem. However, it's essential to ensure that the terminal remains active throughout. Therefore, it is not recommended for projects requiring multiple (often significant) runs of optimization/simulation model (see [batch jobs](#batch-jobs)). To initiate an interactive session, you can use the following command:

To start an interactive session, you can use the following command:
```
srun -N 1 --ntasks-per-node=4  --mem-per-cpu=1gb -t 1:00:00 -p interactive --pty bash
```
Flag `-p` specifies the partition you want to use. More info on interactive jobs can be found [here](https://www.msi.umn.edu/content/interactive-queue-use-srun).

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
export MODULEPATH=$MODULEPATH:~/module-files
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
Job arrays provide a convenient method for scheduling a job that involves executing a code repeatedly with varying parameter values. While a for loop can serve this purpose, it may not always be practical, especially for computationally intensive tasks. This could result in the need to request nodes for extensive periods (or memory), which may exceed the node's usage limit. In such scenarios, job arrays prove beneficial as they enable running the same job multiple times with distinct parameters across different nodes, each assigned a unique job ID.

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
export MODULEPATH=$MODULEPATH:~/module-files
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
The concept discussed in this section should not be confused with a job array. While job arrays facilitate running the same job on different nodes with unique job IDs, here we are trying to run a single job with parallel execution of its components, specifically certain parts of the code. For example, in decomposition algorithms, this approach can help parallelize subproblems.

In Julia, the `Distributed` package can be leveraged to parallelize our code or specific sections of it. This section serves as a concise overview of the coding syntax involved. Let's consider a scenario where you wrote a Julia script implementing a decomposition algorithm, and your goal is to parallelize its subproblems. The following code illustrates how to accomplish this:

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

The above code will run the `subproblem` in parallel on 10 different processes. Note that it doesn't have to be the *_subproblems_* in the code; it can be any part of the code that can be parallelized.

> :warning: While it is possible to generate more processes than the number of cores requested, it's generally discouraged. This is due to the processes competing for the same resources, often leading to performance degradation.

Description of different components (functions/macros) used to setup parallel computing:

* `addprocs(n)` - Make available `n` Julia processes/workers. 

* `@everywhere` - Makes a function/module available on all workers. Note that you can have blocks of code that are performed `@everywhere` by using `begin` and `end`.

* `pmap(n->myfunction(n),range)` - Map the value `n`, which takes all values in `range` onto `myfunction`. This is a parallel equivalent of running a for loop over `range` and evaluating the function in every iteration of the `for` loop. Any julia modules/`.jl` files/global variables required by `myfunction` must be made available `@everywhere`.

This is the most basic (not always necessarily the most efficient) way to parallelize your code. Parallelization is a very extensive topic and one should refer the [documentation](https://docs.julialang.org/en/v1/manual/parallel-computing) to learn more about it.

<a name="common-errors"></a>
## Common errors and best practices while running parallel computations in Julia

* The notorious `EOFError`, `Worker xx terminated`, or `ProcessExitedException`—this error can stem from various causes, often not immediately evident from the stack trace. Generally, it indicates that at least one worker or process crashed during the execution of parallelized code. Occasionally, the cause may be as simple as a bug within the code itself; therefore, it's advisable to initially run the function without parallelization to verify this isn't the case.

    This error arises more frequently due to memory exhaustion. To mitigate this, it's recommended to minimize the usage of `@everywhere` and request more cores than strictly required, thereby also acquiring additional memory. Requesting full nodes instead of partial ones and consolidating all computations on a single node, if feasible, can also help avert this error.

* `Error: On worker x: var not defined`: This frequently occurs when one of your pmapped functions attempts to access a variable that hasn't been defined with `@everywhere`. Fortunately, the error message typically specifies the name of the undefined variable, making it easy to rectify.

* When defining a random parameter across multiple processors, avoid using the syntax `@everywhere var=rand()`. This approach executes the `rand` function independently on each processor, resulting in distinct definitions of var on each processor. Instead, utilize the following approach:
    ```
    var=rand()
    @eval @everywhere var=$var
    ```
    which performs the random generation once on main worker, and then makes that value avaialble `@everywhere`.

* BARON isn't using the options I gave it - Julia's BARON package writes a file called `options` to the current active directory and then deletes it when BARON is done running. The problem is that if you are running multiple instances of BARON in parallel, the `options` file may be closed while another parallel run is trying to read it. The easiest way to fix this is to define a temporary folder for each different parallel instance you are running and to change the active directory to that temporary folder at the beginning of the parallel code.

* As a general guideline, it's advisable to confirm that your function operates without errors for a single instance before attempting to parallel map it. Additionally, it's helpful to validate that your script functions correctly with a limited number of processors on your personal computer (or interactive MSI) before submitting it to MSI.

<a name="installing-software-on-MSI"></a>
## Installing software on MSI

### Julia

Some versions of Julia are already installed on MSI. To see which versions are available, you can use the following command: `module avail julia`. If you want to use one of the already installed version, simply run `module load julia/version-number`. If the version you want is not available, you can either submit a ticket to the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk) to request the version you want, or you can install it yourself as follows: 

From you MSI terminal, run the following command to download Julia:
```
wget https://julialang-s3.julialang.org/bin/linux/x64/1.10/julia-1.10.2-linux-x86_64.tar.gz
tar zxvf julia-1.10.2-linux-x86_64.tar.gz
```
> [!CAUTION]
> Make sure to replace the version number with the version you want to install or see the command [here](https://julialang.org/downloads/platform/).

To be able to use julia when submitting a job to MSI, you need to do two more things. First, you need to build a module for using julia. To do so, create a folder in your home directory called `Modules`. Then, create a folder within Modules called `Julia`. Then, within `Julia`, create a file whose name is the version of Julia you downloaded (e.g. `1.10.2`). Inside this file, write the following line of code: 
```
#%Module
prepend-path PATH /path/to/julia/bin
```
Replace the `/path/to/julia/bin` (e.g., `~/software/julia-1.10.2/bin`)  with the path to the `bin` folder of the julia version you downloaded.

### CPLEX

> [!CAUTION] 
> Please update if you think the following information is outdated.

To install CPLEX to your home folder, first download the installation file from the IBM academic initiative website. You want the version for Linux x86-64. If you fail to log into or download from the IBM website, make sure you close all the add blockers or switch browser. Once this file is downloaded, move it to your home directory on MSI. Then, open the MSI terminal and run the installation package using the code `./cplex_studio1210.linux-x86-64.bin` (make sure the version number is correct). Follow the prompts given to install CPLEX. Remember, to use CPLEX in Julia on MSI, you will also need to add the `JuMP` and `CPLEX` packages in Julia. To use CPLEX in Python, you will need to follow the last prompts in CPLEX installation to install it in python and add CPLEX package under your anaconda environment.

### BARON

To install BARON to your home folder, begin by downloading the zipped files from this [link](https://www.minlp.com/baron-downloads). Next, unzip the files and upload them to your home directory on MSI. Ensure to retrieve the BARON license file and drop it within this folder. To use BARON with Julia on MSI, it's necessary to add the `JuMP` and `BARON` packages in Julia and define the following environment variables (add these to the slurm job script):
```
export BARON_EXEC=/path/to/BARON/executable
export PATH=$PATH:/path/to/BARON/license/folder
```
### Gurobi
Gurobi is installed globally on MSI. To use Gurobi, you can load the module using the following command:

`module load gurobi/VersionNumber`

To see all the available versions of Gurobi available on MSI, you can use the following command:

`module avail gurobi`

To request newer versions of Gurobi, submit a ticket to the MSI [helpdesk](https://www.msi.umn.edu/content/helpdesk).

<a name="hsl"></a>
### HSL Package (Linear solvers for IPOPT)
HSL solver is an alternative linear solver that can be used in IPOPT. It has better efficiency in some NLP systems compared to MUMPS. To use the HSL solver, you will have to first apply for the academic license and the download link of the Coin-HSL Full (Stable) version, visit [here](http://www.hsl.rl.ac.uk/ipopt/). Once you get the file, extract and upload it onto MSI. Find the "libcoinhsl.so" in the "lib" folder and rename it as "libhsl.so". Add the library path into your environment or add the following line into your pbs script":

`export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/YourHslFolderPath/lib`

Finally, specify the linear solver that you want to use in IPOPT by defining the 'linear_solver' as "ma27". There are other solvers you can use in the HSL package, visit [here](http://www.hsl.rl.ac.uk/ipopt/). You can also install HSL solver by recompiling IPOPT, visit [here](https://coin-or.github.io/Ipopt/INSTALL.html) 

<a name="issues-with-different-solvers"></a>
## Good practices/common issues with solvers

<a name="baron-1"></a>
### BARON 

- <u>**Ensuring options file is read when using parallel computing**</u>: The BARON package in Julia writes a file named `options` to the current active directory and deletes it upon completion of BARON's execution. However, a potential issue arises when multiple instances of BARON run in parallel: the `options` file may be closed while another parallel run attempts to read it. To address this, a simple solution is to assign a temporary folder for each parallel instance and set the active directory to this temporary location at the outset of the parallel code execution.

- <u>**BARON significantly slow on Mangi**</u>: **[Untested in recent times - probably outdated]** --
BARON has been observed to run significantly slower on the Mangi queue than on a PC or on Mesabi. The current working theory on this is that something is using Intel's MKL library which Intel has intentionally made incompatible with AMD processors (used by Mangi). Adding the following line to your PBS script may help: `export MKL_DEBUG_CPU_TYPE=5 `.

- <u>**Explicitly specifying CPLEX path**</u>: On MSI, BARON usually cannot automatically locate the CPLEX installation and tends to default to using the open-source solver CLP. Therefore, if you have CPLEX installed, ensure to specify the installation directory using the appropriate option so that BARON can utilize it.

<a name="cplex-1"></a>
### CPLEX
- <u>**Node file management**</u>: <span style="color:red">
[Might be outdated!]
</span> -- CPLEX writes information about different nodes in the branch-and-bound tree to what are called node files. By default, CPLEX writes these to memory. However, memory is a precious commodity on MSI nodes, so it is better to write these node files to disc (and more specifically, to scratch space). To do so, add the following options to CPLEX when defining the model in Julia: `CPX_PARAM_NODEFILEIND=3` and `CPX_PARAM_WORKDIR=/scratch.local/myx500`.

<a name="cplex-1"></a>
### Gurobi
Please update if you encounter any issues with Gurobi.

<a name="ipopt-1"></a>
### IPOPT
Please update if you encounter any issues with IPOPT.

<a name="miscellaneous-troubleshooting-tips"></a>
## Miscellaneous troubleshooting tips

- **ERROR message:** 
    
    ```
    ERROR: LoadError: LoadError: IOError: could not spawn /panfs/roc/groups/10/qizh/yourfile/baron-lin64/baron/tmp/jl_N8EEfQ/baron_problem.bar: permission denied (EACCES)
    ```

    **Potential solution:**
    1. Check the rights on the specific file: `ls -l ~/baron-lin64/baron`
    2. Make the file to be executable: `chmod +x  ~/baron-lin64/baron`
    3. Double check the right on the file. It should be `-rwx------`


- **ERROR message:** 

    ```
    OpenBLAS blas_thread_init: pthread_create failed for thread 1 of 8: Resource temporarily unavailable
    OpenBLAS blas_thread_init: RLIMIT_NPROC 16384 current, 16384 max
    ```
    This implies that the number of processes requested exceeds the max limit. Check the number of processes requested using `addprocs`. Also, removing the `JULIA_NUM_THREADS` command from job script would generally resolve the issue esp. if using `pmap` for parallelization as of version 1.5.0. 

    > **Note**

    
    > This can have different implications depending on Julia version. Always check the documentation for the version you are using.
