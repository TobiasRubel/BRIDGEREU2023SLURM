# Job Scheduler Interactive Exercise

This brief exercise is designed to guide you through working on the Nexus cluster. We'll be downloading and running
the CaPS-SA software for building Suffix Array and Longest Common Prefix indexes. This algorithm and program was designed by 
CBCB students Jamshed Khan and Tobias Rubel working under CBCB faculty Rob Patro and Erin Molloy and UMIACS faculty Laxman Dhulipala. This is research code, and while it is well written as far as they go, it has many of the quirks that you'll likely encounter trying to run other scientist's work. Trial and error will get easier with time.


## 1: ssh into cluster

```
ssh <my-username>@nexuscbcb01.umiacs.umd.edu
mkdir slurmexercise
cd slurmexercise
```

## 2: clone the repo and install

Go to `https://github.com/jamshed/CaPS-SA` and follow the instructions for getting the package:

```
git clone https://github.com/jamshed/CaPS-SA.git
cd CaPS-SA/
mkdir build && cd build/
cmake -DCMAKE_INSTALL_PREFIX=../ ..
make install
cd ..
```

*But wait, this doesn't work!!!* 

When we do this we will get an error that looks something like:

```
CMakeFiles/caps_sa.dir/main.cpp.o: In function `read_input(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >&)':
main.cpp:(.text+0x163): undefined reference to `std::filesystem::__cxx11::path::_M_split_cmpts()'
main.cpp:(.text+0x170): undefined reference to `std::filesystem::file_size(std::filesystem::__cxx11::path const&, std::error_code&)'
collect2: error: ld returned 1 exit status
make[2]: *** [src/CMakeFiles/caps_sa.dir/build.make:98: src/caps_sa] Error 1
make[1]: *** [CMakeFiles/Makefile2:128: src/CMakeFiles/caps_sa.dir/all] Error 2
make: *** [Makefile:136: all] Error 2
```

What does this mean? Stop and think for a moment about why this could be (hint: it has to do with the module system), then proceed click the hidden text to see the solution. 
<details> 
<summary>Click Here for spoiler! </summary>
we forgot to load the gcc module, so the default version of gcc (8.5.0) is being used. We need a newer gcc! Load a newer version then try again. You'll need to delete the contents of `build` and `external` in order for it to compile correctly.
</details> 

## 3: Try to run the program interactively

*Note: you should never run a program on a submission node. This is maybe the most important rule of the cluster: you can edit text and compile code on submission nodes, but running programs causes problems for everyone else and could result in your privileges being revoked.*

Let's get an interactive session. First, lets make an alias for running interactive jobs:

```
vim ~/.bashrc
alias sinteractive="srun --pty --ntasks=1 --mem=32G --qos=highmem --partition=cbcb --account=cbcb --time 1:00:00 bash
:x
source ~/.bashrc
```

The above edits your `bashrc` which will be loaded whenever you open a bash session. It then makes an alias command called `sinteractive` which uses `srun` to request a job (running `bash`) and then moves you to the node. We then save and close the file with `:x` and `source` it so that the commands are run without having to restart the session. Now when you type `sinteractive` it will execute the aliased command. You can modify the flags as you need. 

Type `sinteractive`. You'll get allocated a node. You can confirm that your job was allocated with `squeue --me` and also in this case by running `lspcu` to see the details of the node you're on. You should be on an `AMD EPYC 7313 16-Core Processor`. Each node in the `highmem` queue has 2TB of ram and a dual socket motherboard supporting two of these processors, so you can request a total of 32 cores (64 hyper-threads). 

Note that we are just using a single core, since `--ntasks=1`. If you're running parallel programs you'll want to change this. Likewise, many of you are on the same node potentially. If you use the `--exclusive` flag it will preclude others from using the node at the same time as you. This is a must for getting reliable timing results, but should not be used otherwise. In general do not request more resources than you need. Be considerate of the other researchers. 

Okay, now let's run `CaPS-SA`! The binary is in `bin/`:

```
bash-4.4$ ./bin/caps_sa 
Usage: CaPS_SA <input_path> <output_path> <(optional)-subproblem-count> <(optional)-bounded-context> <(optional)--pretty-print>
```

Let's run it on `data/ecoli.fa`. 

Now, this may work or it may not. You could get an error telling you an illegal instruction was called. This happens because we compiled the code on the submission nodes, but are running it on compute nodes with a different extended instruction set! `CaPS-SA` is compiled with `cmarch=native`, so it may not run on cpus it was not compiled on. If you run into this, go back to the compile step and recompile in the interactive session.

## 4: set up a submission script

`exit` the interactive session to return to a submission node. Now let's first make a really simple submission script to run the program non-interactively. Try to adapt the above interactive code to create a file called `simple.sbatch` which takes an input and output from the command line and submits a job. Then see my solution.  

<details> 
<summary>Click Here for spoiler! </summary>

in `simple.sbatch` place 
```
#!/bin/bash
#SBATCH --time=00:10:00
#SBATCH --ntasks=1
#SBATCH --mem=10G 
#SBATCH --qos=highmem 
#SBATCH --partition=cbcb 
#SBATCH --account=cbcb
IN=$1
OUT=$2

EXEC=/path/to/executable

$EXEC $IN $OUT
```
</details> 


running this with the arguments as before will give you your result. Yay! 

## Bonus: a fancier setup for running many jobs on lots of data

Plato theorized that the soul was tripartite, consisting of the logos (logical part), thymos (spirited part), and eros (appetitative part). A tripartite submission script setup can make your life much easier when scaling to hundreds or thousands of experiments. In our case, we will use the following scripts: `submit.sh`, `drive.sbatch`, and `run.sh`. 

`submit.sh` will be a shell script which characterizes what data we want to run on. For instance, suppose we have a directory `randomdata` which contains 50 files, 0.txt, 1.txt,...,49.txt. We want to run a program on all 50 of these files. Then we might make it as follows:

```
#!/bin/bash

REPLS=({0..49})

for repl in "${REPLS[@]}"; do
    sbatch drive.sbatch $repl;
done
```

then `drive.sbatch` might look like:

```
#!/bin/bash
#SBATCH --time=00:01:00
#SBATCH --ntasks=1
#SBATCH --mem=1G 
#SBATCH --qos=highmem 
#SBATCH --partition=cbcb 
#SBATCH --account=cbcb
run.sh $1
wait
```

which just sets up the job details and runs our wrapper script on them. The wrapper script `run.sh` would then handle the file directories and run the program:

```
#!/bin/bash

INDIR=randomdata
OUTDIR=randomout
REPL=$1
echo "this is job $1" > $OUTDIR/$1.txt 
```

This program just populates the out directory with a bunch of text files saying which replicate we are running on, it doesn't actually do anything interesting. However, this basic framework is one way of building scalable studies without a lot of overhead to start. It's almost always worth the extra 15 minutes to set things up robustly at first. 


## General Advice: 

- test your experiments on one or two small files before setting up massive jobs.
- request as few resources as possible to keep the cluster uncongested and get your jobs out of the queue faster.
- use `--exclusive` for timing programs. 
- use interactive jobs to verify things are working/debug code. 








