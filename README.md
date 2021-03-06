FluProCad

Introduction
FluProCad is a package with short python and bash scripts which can introduce amino-acid mutations in pdb files for fluorescent proteins, generating Gromacs compatible topologies, for molecular dynamics, extracting 1 or more solution structures for the mutant protein (comparable/equivalent to experimental crystal structure) with additional trajectory analysis.

Software Requirements
- Python
- Modeller
- Gromacs

We will start by extracting the files from the FluProCad tarball file. 

>>>	git clone 
>>>	tar -xzvf FluProCad-2.0.tar.gz
>>>	cd FluProCad-2.0
>>> chmod u+x *.sh

Installing Modeller package
Before we start with the installation, run the following command in your terminal/console to check your system architecture:

>>>	uname -a

This will allow us to install the version of MODELLER supported by your system specific architecture. The currently supported architecture should be one of these options:

1)	Linux x86 PC (e.g. RedHat, SuSe).
2)	IBM AIX OS.
3)	x86_64 (Opteron/EM64T) box (Linux).
4)	Alternative x86 Linux binary (e.g. for FreeBSD).

Next, install the molecular modelling package MODELLER. You will also need to register for a license which is available free of charge for academic non-profit institutions.
Download the generic Unix tarball version and change to the directory where you downloaded the.tar.gz file. Unpack the file with following command:
(replace modeller-v.xx with the MODELLER version you have downloaded. Ex: modeller-9.22):

>>	tar -xvzf modeller-v.xx.tar.gz 

Since MODELLER is used here as a python module, check the version of python installed on your system. Currently any version between 2.3 and 3.7 is supported by MODELLER. 
Now run the following command to add the MODELLER library path into the installation file: 
(replace modeller-v.xx with the MODELLER version you have downloaded. Ex: modeller-9.22):

>>	cd modeller-v.xx
>>	cat Install ../mod_path.dat > Install_new
>>	sed -i "s/mkdir -p/mkdir -p -m 755/" Install_new
>>	chmod u+x Install_new

Run the installation script and answer the questions as prompted. If you make any mistakes, you can re-run the script.

#NOTE: the default installation directory can be changed during installation

>>>	./Install_new
>>>	cd ../


1.	Preparing Input files and Topologies

#NOTE: For using shared installation of MODELLER export the following paths or add them to the bashrc file

>>>	export MODINSTALL="/file-path/modeller9.22"
>>>	export EXECUTABLE_TYPE="x86_64-intel8"
>>>	export PYTHONPATH="${PYTHONPATH}:${MODINSTALL}/modlib:${MODINSTALL}/lib/${EXECUTABLE_TYPE}/${PYTHON_VERSION}"
>>>	export LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:${MODINSTALL}/lib/${EXECUTABLE_TYPE}"


We are now going to set up our first mutation simulation. With this example we will prepare the starting structure for our mutant protein. Firstly, edit the file mutations.dat to add the residue index of the residue you want to mutate and what you want to mutate it into.
<residue_index> <3-letter-Residue_name> <single-letter-ChainID>
Ex:	206 ALA A
146 VAL A

Load the GROMACS and supporting modules on your local system/cluster if necessary:

>>>	module load gcc/9.1.0
>>>	module load hpcx-mpi/2.4.0
>>>	module load gromacs/2020.3

Next, check the GMX prefix compiled for the loaded/installed Gromacs version on your system (Ex: gmx, gmx_mpi, or none). Now, run the script md_setup.sh to prepare the mutant structure and topologies for our simulations.


NOTE: Here we use the Gromcacs version 2020.3 so for every time the scripts prompts asking for the chosen Gromacs versio you can select "2020.x" and "gmx_mpi" for the prefix notation.

>>>	./md_setup.sh <PDB_filename> <output_suffix>

(Ex: For the input PDB files names protein.pdb we will run the following command: )

>>>	./md_setup.sh protein test 

The script will call MODELLER package to generate a new mutant structure called "PDB_filename-suffix.pdb". Further, it will prepare the Gromacs topologies and solvate the protein in a neutral water solvent box with 0.15M concentration. 

You should now have a new directory called <PDB_filename-suffix> with the topologies, forcefield and .mdp files that we will use to start out MD simulations.


2.	Equilibration and Production MD Simulations
We are once again going to use grompp to assemble the structure, topology, and simulation parameters into a binary input file (.tpr). 

If you have access to a supercomputer/supercluster or local cluster, we can use a bash script submit a parallel job as our system is rather large and may need significant computational time.

>>>	cd <PDB_filename-suffix>

Open the file gmx-jobs.sh with a text file editor of your choice or in the terminal. Here you can edit the number of nodes/cores you prefer to use for the job. Edit the name of the Gromacs module corresponding to the modules available on the cluster in use.
Below are the contents of the bash script file which can be used for submitting a batch job on the PUHTI cluster at CSC: 

(#NOTE: IF YOU PREFER USE YOUR LOCAL SYSTEM TO RUN THE SIMULATIONS, REFER TO THE NEXT SECTION)

>>>	vi gromacs_jobs.sh

#!/bin/bash
#SBATCH --job-name=FPC-test
#SBATCH --account=<project_name_CSC>
#SBATCH --partition=large
#SBATCH --time=24:00:00
#SBATCH --nodes=4
#SBATCH --tasks-per-node=24
#SBATCH --mem-per-cpu=4000

# this script runs a 192 core (8 full nodes) gromacs job.
export OMP_NUM_THREADS=1

module load gcc/9.1.0
module load hpcx-mpi/2.4.0
module load gromacs/2020.3 	  #change if you want a different version

gmx_mpi grompp -f em.mdp -p topol.top -c ions.gro -o em
srun gmx_mpi mdrun -deffnm em

gmx_mpi grompp -f nvt.mdp -p topol.top -c em.gro -r em.gro -o nvt
srun gmx_mpi mdrun -deffnm nvt

gmx_mpi grompp -f npt.mdp -p topol.top -c nvt.gro -t nvt.cpt -o npt
srun gmx_mpi mdrun -deffnm npt

gmx_mpi grompp -f md.mdp -p topol.top -c npt.gro -t npt.cpt -o topol
srun gmx_mpi mdrun -s topol -dlb yes

Here is an example for submitting a SLURM/PBS batch job on a cluster. (Please check the correct of commands for submitting a batch job if your cluster supports only PBS commands.)

>>>	sbatch gromacs_jobs.sh

# To check the status of the submitted job use the following command:

>>> squeue -u <CSC-username>


For the following commands replace <gmx_version> by the prefix compiled for the GROMACS version installed on your system.

We will run the energy minimization first:

>>>	<gmx_prefix> grompp -f em.mdp -p topol.top -c ions.gro -o em.tpr
>>>	<gmx_prefix> mdrun -deffnm em

Then continue with a 2-phase equilibration step:

>>>	<gmx_prefix> grompp -f nvt.mdp -c em.gro -r em.gro -p topol.top -o nvt.tpr
>>>	<gmx_prefix> mdrun -deffnm nvt


>>>	<gmx_prefix> grompp -f npt.mdp -p topol.top -c nvt.gro -r nvt.gro -t nvt.cpt -o npt.tpr
>>>	<gmx_prefix> mdrun -deffnm npt

We will now start the final production run of 100ns:

>>>	<gmx_prefix> grompp -f md.mdp -p topol.top -c npt.gro -r npt.gro -t npt.cpt -o topol.tpr
>>>	<gmx_prefix> mdrun -s topol

 
3.	Analysis 
Now that we have run the MD simulations, let’s run some analysis on the system in the 100ns trajectory. As we want to extract solution structure(s) from the trajectory that represent the ensemble generated with MD, we will use a set of RMSD and clustering analysis (GROMOS algorithm) tools available in GROMACS. 
Check if you are in the directory with all the trajectory files generated from the MD production run. The following command will run the analysis on the trajectories:


>>>	sbatch analyze-jobs.sh <output-tag>

Once the job is completed you can find all the outputs from the above script in a new sub-directory called MD-analysis. Have a look at the ‘.xvg’ files to check for the structural stability (RMSD) and flexible residue regions (RMSF).

The directory should also include clustering output log and pdb files. The log file summarizes the number of clusters found in the trajectory and extracts a single representative structure for each cluster into the pdb file.

Clusters with reasonable population size (>200 frames) and distribution are considered as the representative solution structures for the selected mutation.

With your choice of visualization program (PyMol, VMD,..), now open the PDB file (clusters-0.30-<output-suffix>.pdb) with cluster centres of different populations extracted from the MD-trajectory.






 
