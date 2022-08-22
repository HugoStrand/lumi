# Lumi notes

- https://docs.lumi-supercomputer.eu/

## Example job-script

'''
#!/bin/bash -l
#SBATCH --job-name=test-job
#SBATCH --account=project_465000175
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=128
#SBATCH --cpus-per-task=1
#SBATCH --partition=standard

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun hostname
'''