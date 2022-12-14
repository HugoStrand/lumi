# Lumi Installation Notes

- https://docs.lumi-supercomputer.eu/

## Example job-script

```bash
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
```

## Modules

```bash
ml craype-x86-milan
ml LUMI/22.08 partition/C

ml PrgEnv-gnu
ml cray-fftw
ml cray-hdf5
ml cray-python

ml Boost/1.79.0-cpeGNU-22.08
ml GMP/6.2.1-cpeGNU-22.08
```

The `cray-python` comes with `mpi4py`, as can be seen from:
```bash
hustrand@uan03:~/dev/triqs/build> python -c "import mpi4py; print(mpi4py.__file__)"
/opt/cray/pe/python/3.9.12.1/lib/python3.9/site-packages/mpi4py/__init__.py
```

# Install TRIQS

## Install extra Python packages

Need to install some python modules by hand

```bash
pip install --user h5py
pip install --user mako
```

## Clone GitHub repository

Note, using special branch with fixes for `cray-mpich`. ([This pull request](https://github.com/TRIQS/triqs/pull/857) might eventually be merged with the `3.1.x` branch.)
```
git clone https://github.com/TRIQS/triqs.git --branch prq_cray_mpich_support
```

## CMake

Generate make files using `cmake`
```bash
CXX=CC CC=cc cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/apps/triqs \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-Ofast -funroll-loops" \
  ..
```

## Fix `TRIQS/mpi` 

Remove default `TRIQS/mpi` source `deps/mpi_src` downloaded by `cmake` with the branch with fixes for `cray-mpich`
```
rm -rf deps/mpi_src
git clone  https://github.com/TRIQS/mpi.git --branch prq_cray_mpich_support deps/mpi_src
```

For details see [this pull request](https://github.com/TRIQS/mpi/pull/10).

## Build

Build TRIQS using `make` using 128 parallell processes
```bash
make -j 128
```

## Test

The Triqs test suite can be run from the login node by starting an interactive jobsession
```bash
salloc --nodes=1 --partition=standard --account=project_465000175 --time=00:05:00
ctest
```

With this setup all tests passes:
```bash
...
100% tests passed, 0 tests failed out of 251

Total Test time (real) = 120.03 sec
```

Note that the tests can not be executed in a shell on the compute note
```
## srun --cpu_bind=none --nodes=1 --pty bash -i ## DO NOT START SHELL ON NODE
```
since `Triqs` runs `srun` internally for the mpi based tests.

## Install

To install to the `CMAKE_INSTALL_PREFIX` directory (above `$HOME/apps/triqs`) run
```bash
install -j 128
```

## Production

Setup environment variables for Triqs using:
```
source /users/hustrand/apps/triqs/share/triqs/triqsvars.sh
```

To test that the environment is actually working use
```
python -c "import triqs; print(triqs.__file__)"
```

# Install TRIQS/cthyb

## Clone the GitHub repository
```
git clone https://github.com/TRIQS/cthyb.git --branch 3.1.x
```

## CMake command

```bash
CXX=CC CC=cc cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/apps/cthyb \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-Ofast -funroll-loops" \
  ..
```

## Build
```
make -j 128
```

## Test

Standing in the cmake build directory, the `cthyb` test suite can be run from the login node by starting an interactive jobsession
```bash
salloc --nodes=1 --partition=standard --account=project_465000175 --time=00:05:00
ctest
```

With this setup all test passes:
```
...
100% tests passed, 0 tests failed out of 26

Total Test time (real) =  72.38 sec
```

## Install

To install to the `CMAKE_INSTALL_PREFIX` directory (above `$HOME/apps/cthyb`) run
```bash
install -j 128
```

## Usage

Before using in production, make sure to source the cthyb environment variables before running
```
source /users/hustrand/apps/cthyb/share/triqs_cthyb/triqs_cthybvars.sh
```

To test that the environment is actually working use
```
python -c "import triqs_cthyb; print(triqs_cthyb.__file__)"
```

# RSPt (with linking to `rspt_extsol`)

## Clone the GitHub repositories

```
cd ~/dev
git clone https://github.com/HugoStrand/rspt_extsol.git
git clone git@github.com:uumaterialstheory/rspt.git --branch dev_extsol
```

## Configure and build `rspt_extsol`

```
cd ~/dev/rspt_extsol
mkdir build
cd build
CXX=CC CC=cc cmake \
  -DCMAKE_CXX_FLAGS="-static-libstdc++" \
  ..
make
```

## `RSPTmake.inc`

Create the file `RSPTmake.inc` with the contents:
```
#
# Cray/HPE PrgEnv-gcc @ LUMI 22.08
#

FHOME            =
FCOMPILER        = ftn -std=legacy
FCOMPILERFLAGS   = -O2 -ffree-line-length-0
FCPPFLAGS        = -DMPI -DNOMPIMOD -DEXTERNAL_CTHYB
FTARGETARCH      =
FORTRANLIBS      = -static-libstdc++ \
-L/opt/cray/pe/python/3.9.12.1/lib/ -Wl,-rpath=/opt/cray/pe/python/3.9.12.1/lib/ -lpython3.9 \
-L$(HOME)/dev/rspt_extsol/build/ -Wl,-rpath=$(HOME)/dev/rspt_extsol/build/ -lrspt_extsol
F90COMPILER      = ftn
F90COMPILERFLAGS = $(FCOMPILERFLAGS) -ffree-form

# gcc
CCOMPILER        = cc -static-libstdc++
CCOMPILERFLAGS   = -O2
CTARGETARCH      = 
CPPFLAGS         = -DMPI
CLOADER          = 

## LIBRARIES AND INCLUDE DIRECTORIES
LAPACKLIB        =
BLASLIB          =
FFTWLIB          =
EXTRALIBS        = -lfftw3 -lmpi -lmpifort 
INCLUDEDIRS      = 

EXEC             = rspt
```

Note that using higher optimizations than `-O2` cause RSPt to produce inconsistent results for the `pyrspt` tests below.

## Build

```
make
```

# `pyrspt`

## Install additinal Python module `ase`

Install `ase` the Atomic Simulation Environment (ASE) and `pydlr` the Discrete Lehman Representation Python modules.
```
pip install --user ase
pip install --user pydlr
```

## Clone the GitHub repository

```
git clone https://github.com/HugoStrand/pyrspt.git
```

## Setup environment

Add to `~/.profile`
```bash
export PYTHONPATH=$HOME/dev/pyrspt/:$PYTHONPATH
export RSPT_BIN=$HOME/dev/rspt/bin
export RSPT_NK=1
export RSPT_NB=1
export RSPT_CMD="rspt"
```

## Test

```bash
cd ~/dev/pyrspt/pyrspt/test
nosetests
```

The test passes
```
hustrand@uan04:~/dev/pyrspt/pyrspt/test> nosetests
..
----------------------------------------------------------------------
Ran 2 tests in 53.404s

OK
```

## Production

In the job script set the environment variables according to the wanted MPI parallelization. Note that `RSPT_CMD` has to be changed to `srun rspt`.
```bash
...

export RSPT_NK=12
export RSPT_NB=2
export RSPT_CMD="srun rspt"

python your_pyrspt_calculation_script.py
```

