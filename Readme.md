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

Replace `/users/hustrand` with `$HOME`
```
#
# Cray/HPE PrgEnv-gcc @ LUMI 22.08
#
## COMPILERS
# 
FHOME            =
FCOMPILER        = ftn -std=legacy
#FCOMPILERFLAGS   = -Ofast -funroll-loops -ffree-line-length-0
FCOMPILERFLAGS   = -O2 -ffree-line-length-0
FCPPFLAGS        = -DMPI -DNOMPIMOD -DEXTERNAL_CTHYB
FTARGETARCH      =
FORTRANLIBS      = \
-static-libstdc++ \
-L/opt/cray/pe/gcc/11.2.0/snos/lib64/ -lgfortran -lmpi -lmpifort \
-L/opt/cray/pe/hdf5/1.12.1.5/gnu/9.1/lib/ \
-Wl,-rpath=/opt/cray/pe/hdf5/1.12.1.5/gnu/9.1/lib/ \
-lhdf5 \
-L/users/hustrand/dev/rspt_extsol/build/ \
-Wl,-rpath=/users/hustrand/dev/rspt_extsol/build/ \
-lrspt_extsol \
-L/opt/cray/pe/python/3.9.12.1/lib/ \
-Wl,-rpath=/opt/cray/pe/python/3.9.12.1/lib/ \
-lpython3.9 

F90COMPILER      = ftn
F90COMPILERFLAGS = $(FCOMPILERFLAGS) -ffree-form
# gcc
CCOMPILER        = cc -static-libstdc++
#CCOMPILERFLAGS   = -Ofast -funroll-loops 
CCOMPILERFLAGS   = -O2
CTARGETARCH      = 
CPPFLAGS         = -DMPI
CLOADER          = 

## LIBRARIES AND INCLUDE DIRECTORIES
LAPACKLIB        =
BLASLIB          =
FFTWLIB          =
EXTRALIBS        = -lfftw3 
INCLUDEDIRS      = 

EXEC             = rspt
```

## Build

```
make
```

# `pyrspt`

## Install additinal Python module `ase`

Install the Atomic Simulation Environment (ASE)
```
pip install --user ase
```

## Clone the GitHub repository

```
git clone https://github.com/HugoStrand/pyrspt.git
```

## Setup environment

Add to `~/.profile`
```
export PYTHONPATH=$HOME/dev/pyrspt/:$PYTHONPATH
export RSPT_BIN=$HOME/dev/rspt/bin
export RSPT_NK=1
export RSPT_NB=1
export RSPT_CMD="rspt"
```

## Test

```
cd ~/dev/pyrspt/pyrspt/test
nosetests
```

The Al volume calculation fails with a 5% error in the equilibrium lattice parameter...
```
hustrand@uan01:~/dev/pyrspt/pyrspt/test> nosetests
FF
======================================================================
FAIL: test_Al_vol.test_vol
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/opt/cray/pe/python/3.9.12.1/lib/python3.9/site-packages/nose/case.py", line 198, in runTest
    self.test(*self.arg)
  File "/pfs/lustrep3/users/hustrand/dev/pyrspt/pyrspt/test/test_Al_vol.py", line 62, in test_vol
    np.testing.assert_almost_equal(a0, 4.010854032643257, decimal=5)
  File "/opt/cray/pe/python/3.9.12.1/lib/python3.9/site-packages/numpy/testing/_private/utils.py", line 597, in assert_almost_equal
    raise AssertionError(_build_err_msg())
AssertionError: 
Arrays are not almost equal to 5 decimals
 ACTUAL: 4.065625144181517
 DESIRED: 4.010854032643257
-------------------- >> begin captured stdout << ---------------------
a = 3.80, E = -6585.952382769626
a = 3.88, E = -6586.040048099434
a = 3.96, E = -6586.090516231364
a = 4.04, E = -6586.110659273062
a = 4.12, E = -6586.106759719129
a = 4.20, E = -6586.084856066468
V_0 = 16.800492365113197 A^3, E_0 = -6586.1117096731205 eV, B = 59.92687653620175 GPa
a0 = 4.065625144181517

--------------------- >> end captured stdout << ----------------------

======================================================================
FAIL: test_tetra_fcc_Al_different_lengthscales.test_tetra_fcc_Al_different_lenghtscales
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/opt/cray/pe/python/3.9.12.1/lib/python3.9/site-packages/nose/case.py", line 198, in runTest
    self.test(*self.arg)
  File "/pfs/lustrep3/users/hustrand/dev/pyrspt/pyrspt/test/test_tetra_fcc_Al_different_lengthscales.py", line 38, in test_tetra_fcc_Al_different_lenghtscales
    np.testing.assert_almost_equal(
  File "/opt/cray/pe/python/3.9.12.1/lib/python3.9/site-packages/numpy/testing/_private/utils.py", line 597, in assert_almost_equal
    raise AssertionError(_build_err_msg())
AssertionError: 
Arrays are not almost equal to 7 decimals
 ACTUAL: -6585.817597293285
 DESIRED: -6585.641697576646
-------------------- >> begin captured stdout << ---------------------
E = -6585.817597293285 eV, ef = 9.035805147078616 eV
E = -6585.641697576646 eV, ef = 8.56671393239479 eV

--------------------- >> end captured stdout << ----------------------

----------------------------------------------------------------------
Ran 2 tests in 50.262s

FAILED (failures=2)
```

The input files only differ in the choice of `lengthscale` vs. `latticevectors`
```
hustrand@uan04:~/dev/pyrspt/pyrspt/test> cat rspt_calc_Al_2022-08-25_18\:55\:*/sym/symt.inp
########################################################################
#
# RSPt symt.inp file generated using ASE 3.22.1 and the Atoms object
# 
# {'cell': array([[0.     , 2.03625, 2.03625],
#       [2.03625, 0.     , 2.03625],
#       [2.03625, 2.03625, 0.     ]]),
# 'numbers': array([13]),
# 'pbc': array([ True,  True,  True]),
# 'positions': array([[0., 0., 0.]])}
#
# Unit cell volume: 16.88586401953125 A^3 = 113.95145885323446 a_0^3
#
########################################################################

lengthscale
  4.8481192814688034 

latticevectors
   0.0000000000000000  0.7937005259840997  0.7937005259840997 
   0.7937005259840997  0.0000000000000000  0.7937005259840997 
   0.7937005259840997  0.7937005259840997  0.0000000000000000  

atoms
1
  0.0000000000000000  0.0000000000000000 -0.0000000000000000  13  l     a   # Al 

spinaxis
 0 0 0  c

########################################################################
#
# RSPt symt.inp file generated using ASE 3.22.1 and the Atoms object
# 
# {'cell': array([[0.     , 2.03625, 2.03625],
#       [2.03625, 0.     , 2.03625],
#       [2.03625, 2.03625, 0.     ]]),
# 'numbers': array([13]),
# 'pbc': array([ True,  True,  True]),
# 'positions': array([[0., 0., 0.]])}
#
# Unit cell volume: 16.88586401953125 A^3 = 113.95145885323446 a_0^3
#
########################################################################

lengthscale
  3.8479548237354453 

latticevectors
   0.0000000000000000  1.0000000000000000  1.0000000000000000 
   1.0000000000000000  0.0000000000000000  1.0000000000000000 
   1.0000000000000000  1.0000000000000000  0.0000000000000000  

atoms
1
  0.0000000000000000  0.0000000000000000 -0.0000000000000000  13  l     a   # Al 

spinaxis
 0 0 0  c

```

The problem is in how `bz/cub.inp` is generated

```
hustrand@uan04:~/dev/pyrspt/pyrspt/test> cat rspt_calc_Al_2022-08-25_18\:55\:*/bz/cub.inp 
# R
  .000000000000000  .793700525984099  .793700525984099
  .793700525984099  .000000000000000  .793700525984099
  .793700525984099  .793700525984099  .000000000000000
# M
     0     4     4
     4     0     4
     4     4     0
# R
  .000000000000000  1.00000000000000  1.00000000000000
  1.00000000000000  .000000000000000  1.00000000000000
  1.00000000000000  1.00000000000000  .000000000000000
# M
     0     1     1
     1     0     1
     1     1     0
```

Testing to rebase `dev_extsol` on current `master`... Seems to work

## Production

In the job script set the environment variables according to the wanted MPI parallelization

```
...

export RSPT_NK=12
export RSPT_NB=2
export RSPT_CMD="srun rspt"

python your_pyrspt_calculation_script.py
```

