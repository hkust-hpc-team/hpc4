Migration from HPC3 to HPC4
===============================================

Overview
--------
This guide will assist you in migrating your work from the HPC3 system to the new HPC4 system. The process involves several key steps:

1. Transferring your data and code from HPC3 to HPC4
2. Accessing the new HPC4 system
3. Compiling your code on HPC4
4. Testing your code on HPC4
5. Running large jobs on HPC4 using SLURM

Each step is crucial for ensuring a smooth transition to the new system.

While this guide aims to make the process as straightforward as possible, if you encounter any issues, please contact the ITSC Helpdesk at `cchelp@ust.hk <mailto:cchelp@ust.hk>`_ for assistance.

Step 1: Transfer Data
---------------------
In this step, you will transfer your files from HPC3 to HPC4. This process can be conceptualized as relocating your digital workspace from one computing environment to another. We will utilize two primary tools:

- ``sshfs``: This tool allows access to HPC3's files as if they were locally present on HPC4.
- ``fpsync``: This tool efficiently copies files from one directory to another by grouping files into batches and synchronizing them in parallel.

The following procedure outlines how to copy files from your HPC3 home directory:

.. code-block:: bash

   # Step 1: Create a directory on HPC4 to access HPC3 files
   mkdir ~/hpc3-storage 

   # Step 2: Connect to HPC3's file system
   sshfs $USER@hpc3.ust.hk:/home/$USER ~/hpc3-storage

   # Step 3: Copy files from HPC3 to HPC4
   # This command will copy all files, displaying progress (-vv) and using 16 parallel processes (-n 16)
   fpsync -vv -f 1024 -n 16 ~/hpc3-storage/my-source-code/ ~/my-source-code/

   # Step 4: Disconnect from HPC3's file system when finished
   fusermount -u ~/hpc3-storage && rmdir ~/hpc3-storage

For larger datasets stored in the scratch space, employ a similar process:

.. code-block:: bash

   # Create a mounting point for HPC3 scratch space
   mkdir ~/hpc3-scratch

   # Connect to HPC3's scratch space
   sshfs $USER@hpc3.ust.hk:/scratch/PI/$USER ~/hpc3-scratch

   # Copy datasets to HPC4's scratch space
   fpsync -vv -f 2048 -s 10G -n 16 /home/$USER/hpc3-scratch/path/to/hpc3/dataset/ /scratch/$USER/my-hpc3-dataset/

   # Disconnect when finished
   fusermount -u ~/hpc3-scratch && rmdir ~/hpc3-scratch

Note: Replace ``PI`` with your Principal Investigator's username if necessary.

Important considerations:
- The ``-vv`` option displays detailed progress of the file transfer.
- The ``-f 2048`` option sets the maximum number of files per batch to 2048.
- The ``-s 10G`` option sets the approximate total file size per batch to 10GB.
- The ``-n 16`` option utilizes 16 parallel processes for faster copying. Please do not adjust this option.

Upon completion of these steps, your code and datasets will be available on the HPC4 system, prepared for subsequent stages of compilation and execution.

Step 2: Interactive Session
---------------------------
Use SLURM to access a node of the correct CPU type.

For AMD node (256 cores):

.. code-block:: bash

   srun -A jiy -p cpu -C amd --nodes=1 --ntasks-per-node=1 --cpus-per-task=256 --pty bash

For Intel node (128 cores):

.. code-block:: bash

   srun -A jiy -p cpu -C intel --nodes=1 --ntasks-per-node=1 --cpus-per-task=128 --pty bash

Step 3: Compile Code
--------------------
Use the appropriate compiler based on the CPU type.

For AMD:

.. code-block:: bash

   # Load AOCC compiler
   module load aocc

   # Compile example
   clang -O3 -march=native -mtune=native -fopenmp main.c -o main_amd

For Intel:

.. code-block:: bash

   # Load Intel compiler
   module load intel/oneapi-2023

   # Compile example
   icc -O3 -march=native -mtune=native -qopenmp main.c -o main_intel

Step 4: Test Small Case
-----------------------
Run a small test directly on the compiling node.

Example:

.. code-block:: bash

   # Set OpenMP threads
   export OMP_NUM_THREADS=4

   # Run the compiled program
   ./main_amd  # or ./main_intel

   # Check the output
   cat output.txt

Step 5: Large Run with SLURM
----------------------------
Use ``sbatch`` command with a script for larger runs.

Create a SLURM job script (``job.sh``):

.. code-block:: bash

   #!/bin/bash

   #SBATCH --job-name=my-hpc4-job
   #SBATCH --nodes=1
   #SBATCH --ntasks-per-node=1
   #SBATCH --cpus-per-task=256
   #SBATCH --partition=cpu
   #SBATCH --constraint=amd
   #SBATCH --time=1-0:0:0
   #SBATCH --account=my-account
   #SBATCH --mail-user=username@ust.hk
   #SBATCH --mail-type=begin,end

   set -x

   # Load necessary modules
   module load aocc

   # Set environment variables
   export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

   # Run the program
   ./main_amd > output_large.txt

Submit the job:

.. code-block:: bash

   sbatch job.sh

Check job status:

.. code-block:: bash

   squeue -u $USER

After job completion, check the output:

.. code-block:: bash

   cat output_large.txt
   cat slurm-<job_id>.out  # For SLURM output and errors