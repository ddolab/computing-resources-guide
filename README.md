# Getting Started with MSI

This documentation is designed to assist you in getting started with MSI by providing guidance on accessing the system, submitting jobs, and adhering to best practices. You are encouraged to supplement this information with additional useful information (or removal of outdated information) as needed in the future.

## Table of Contents

1. [When to use MSI?](#when-to-use-MSI)
2. [Modules](#modules)
2. [Running jobs](#running-jobs)
    2.1 [Interactive Jobs](#interactive-jobs)
    2.2 [Batch Jobs](#batch-jobs)
    2.3 [Job Arrays](#job-arrays)
3. [Parallelizing part of your script using pamp]
(#parallelizing-part-of-your-script-using-pamp)

<a name="when-to-use-MSI"></a>
## When to use MSI?
Consider using MSI over your PC in the following cases:

- Large optimization problems or simulations requiring extensive computational time.
- Requirement for accessing more memory than available on your PC.
- Running computational experiments with a huge number of instances.
- Running parallel computations that require more cores than available on your PC.

MSI has the follwoing partitions available for use:
- Agate
- Mangi
- Mesabi [soon to be discontinued]

Each partition has a different architecture and more information on that can be found [here](https://www.msi.umn.edu/partitions).

<a name="modules"></a>
## Modules
Modules are softwares that are either installed on the MSI system globally (available to all users) or can be locally installed by the user on their own profile. For example, Gurobi is pre-installed on MSI and can be loaded using the following command:
```
module load gurobi/10.0.1
```
The above command loads Gurobi version 10.0.1. To see all the available versions of Gurobi available on MSI, you can use the following command:
```
module avail gurobi
```
For newer versions of globally installed modules, you need to contact the MSI staff. Also, for globally installed modules that require a license (e.g. Gurobi), you need to contact the MSI staff to grant you the access.

The locally installed softwares needs to be placed in a particular folder and the user needs to add that folder to their PATH variable. For example, if you have installed a software in the folder `~/software`, you can add the following line to your `.bashrc` file:
```
export PATH=$PATH:~/software
```

<a name="running-jobs"></a>
## Running Jobs 

### Interactive Jobs
Interactive sessions are useful for debugging and testing your code or maybe if you only have to run a particular instance of optimization/simulation problem. To start an interactive session, you can use the following command:
```
srun -N 1 --ntasks-per-node=4  --mem-per-cpu=1gb -t 1:00:00 -p interactive --pty bash
```

More info on interactive jobs can be found [here](https://www.msi.umn.edu/content/interactive-queue-use-srun).

### Batch Jobs

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

More info on job scheduling using Slurm can be found [here](https://www.msi.umn.edu/content/job-submission-and-scheduling-slurm).

### Job Arrays
Job arrays are useful when you have to run the same job multiple times with different parameters. For example, suppsoe you have the following Julia script that solves an optimization problem: 

```
using JuMP, Gurobi

function solve(a)
    m = Model(Gurobi.Optimizer)
    @variable(m, x>=0)
    @variable(m, y>=0)
    @constraint(m, x + y == a)
    @objective(m, Min, 2x + y)
    optimize!(m)
    return value(x), value(y)
end

pritnln(solve(ARGS[1]))
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

## Parallelizing part of your script using pamp
This is not the same as job arrays. The main difference is that in Job arrays you are running the same job on different nodes (with different job IDs), whereas here it's a single job with part of it running in parallel. For example, in decomposition algorithms, you can use this to parallelize the subproblems.

Assume you have a Julia script that solves a large optimization problem and you want to parallelize the subproblems. You can use the following code:

```
using Distributed

addprocs(10)

@everywhere function parallelizable_part(x)
    sleep(10)
    return x^2
end

z = pmap(parallelizable_part, 1:10; retry = 3)
```

The above code will run the `parallelizable_part` function in parallel on 10 different processes.

> :warning: Although you can create more processes than the number of cores requested, it's not recommended. This is because the processes will be competing for the same resources and the performance will likely degrade.