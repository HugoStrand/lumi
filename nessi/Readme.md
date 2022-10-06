
# Building NESSi on LUMI

## Module environment

```bash
ml purge
ml LUMI/22.08 partition/C

ml cpeGNU
ml cray-fftw
ml cray-hdf5
ml cray-python

ml Boost/1.79.0-cpeGNU-22.08
ml GMP/6.2.1-cpeGNU-22.08
ml Eigen/3.4
```

```
~/> ml

Currently Loaded Modules:
  1) init-lumi/0.1                        (S)  12) cray-mpich/8.1.18         23) ncurses/6.2-cpeGNU-22.08
  2) ModuleLabel/label                    (S)  13) cray-libsci/22.08.1.1     24) gettext/0.21-cpeGNU-22.08-minimal
  3) LUMI/22.08                           (S)  14) cray-dsmml/0.2.2          25) XZ/5.2.5-cpeGNU-22.08
  4) craype-x86-milan                          15) perftools-base/22.06.0    26) lz4/1.9.3-cpeGNU-22.08
  5) craype-accel-host                         16) cpeGNU/22.08              27) zstd/1.5.2-cpeGNU-22.08
  6) libfabric/1.15.0.0                        17) cray-fftw/3.3.10.1        28) ICU/71.1-cpeGNU-22.08
  7) craype-network-ofi                        18) cray-hdf5/1.12.1.5        29) Boost/1.79.0-cpeGNU-22.08
  8) xpmem/2.4.4-2.3_9.1__gff0e1d9.shasta      19) cray-python/3.9.12.1      30) GMP/6.2.1-cpeGNU-22.08
  9) partition/C                          (S)  20) bzip2/1.0.8-cpeGNU-22.08  31) Eigen/3.4
 10) gcc/11.2.0                                21) zlib/1.2.12-cpeGNU-22.08
 11) craype/2.7.17                             22) gzip/1.12-cpeGNU-22.08

  Where:
   S:  Module is Sticky, requires --force to unload or purge
```

## Clone NESSi

```bash
~/> cd dev
~/dev> git clone https://github.com/nessi-cntr/nessi.git
~/dev> cd nessi/libcntr
```

## Fix Eigen3 include path

Fix for relative include pattern of `Eigen3`
```
~/dev/nessi/libcntr> mkdir eigen_include
~/dev/nessi/libcntr> ln -s /appl/lumi/SW/LUMI-22.08/common/EB/Eigen/3.4/include eigen_include/eigen3
```

Fix for HP/Cray MPICH and Find_MPI of cmake. Edit `./nessi/libcntr/CMakeLists.txt` according to
```
diff --git a/libcntr/CMakeLists.txt b/libcntr/CMakeLists.txt
index 9b913c7..d12e002 100644
--- a/libcntr/CMakeLists.txt
+++ b/libcntr/CMakeLists.txt
@@ -18,7 +18,7 @@ endif ()
 option(mpi "Build with OpenMPI support" OFF)
 if (mpi)
     message(STATUS "Building with OpenMPI")
-    find_package(MPI REQUIRED)
+    find_package(MPI REQUIRED CXX)
     add_definitions("-DCNTR_USE_MPI")
     set(CMAKE_C_COMPILE_FLAGS "${CMAKE_C_COMPILE_FLAGS} ${MPI_COMPILE_FLAGS}")
     set(CMAKE_CXX_COMPILE_FLAGS
```

## Configure and build NESSi

Command line for `cmake` stored in the file `cmake.sh` (and made executable using `chmod +x cmake.sh`)
```
CC=cc CXX=CC \
  cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/apps/libcntr \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_DOC=OFF \
  -DBUILD_TEST=NO \
  -Domp=ON \
  -Dhdf5=ON \
  -Dmpi=ON \
  -DCMAKE_INCLUDE_PATH="./eigen_include" \
  ..
```

```bash
~/dev/nessi/libcntr> mkdir cbuild
~/dev/nessi/libcntr> cd cbuild
~/dev/nessi/libcntr/cbuild> ../cmake.sh
~/dev/nessi/libcntr/cbuild> make -j 128
~/dev/nessi/libcntr/cbuild> make install
```

## Configure and build NESSi example programs

```bash
./nessi/libcntr/cbuild> cd ../../examples
```

```
CC=cc CXX=CC \
  cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/apps/libcntr_examples \
  -DCMAKE_BUILD_TYPE=Release \
  -Domp=ON \
  -Dhdf5=ON \
  -Dmpi=ON \
  -DCMAKE_INCLUDE_PATH="~/apps/libcntr/include;../libcntr/eigen_include" \
  -DCMAKE_LIBRARY_PATH="~/apps/libcntr/lib" \
  ..
```

```bash
~/dev/nessi/examples> mkdir cbuild
~/dev/nessi/examples/cbuild> ../cmake.sh
~/dev/nessi/examples/cbuild> make -j 128
~/dev/nessi/examples/cbuild> cd ..
```

## Setup and run `demo_gw.py`

Add the NESSi python module and the NESSi library to your shell environment
```
export PYTHONPATH=$HOME/dev/nessi/libcntr/python3:$PYTHONPATH
export LD_LIBRARY_PATH=$HOME/apps/libcntr/lib:$LD_LIBRARY_PATH
```

Change MPI run command from `mpirun` to `srun` in `./nessi/examples/utils/demo_gw.py`
```bash
~/dev/nessi/examples> sed -i bak 's/mpirun/srun/g' ./utils/demo_gw.py
~/dev/nessi/examples> git diff 
```

```
diff --git a/examples/utils/demo_gw.py b/examples/utils/demo_gw.py
index 77d1a65..d4b5f5c 100644
--- a/examples/utils/demo_gw.py
+++ b/examples/utils/demo_gw.py
@@ -122,7 +122,7 @@ if __name__ == '__main__':
     GenField(E0,omega,Np,Nt,h,file_field)
     
     output_file = 'out/gw_'
-    mpicmd = 'mpirun -n 2'
+    mpicmd = 'srun -n 2'
     Run(sysparams,solverparams,file_field,output_file,mpicmd,runpath='./')
```

Start an interactive allocation on LUMI
```bash
~/dev/nessi/examples> salloc --nodes=1 --ntasks=128 --partition=debug --account=project_465000175 --time=00:05:00

salloc: Pending job allocation 1748084
salloc: job 1748084 queued and waiting for resources
salloc: job 1748084 has been allocated resources
salloc: Granted job allocation 1748084
```

Run `demo_gw.py`
```
~/dev/nessi/examples> python ./utils/demo_gw.py

************************************************************
   Test program: Hubbard 1d in GW approximation
************************************************************

 reading input file ...

--------------------------------------------------
 Estimation of memory requirements
--------------------------------------------------
Total
Hamiltonian : 1 MB
Propagators : 1 MB
Per rank
Hamiltonian : 0.5 MB
Propagators : 0.5 MB
--------------------------------------------------


--------------------------------------------------
     Solution of equilibrium problem 
--------------------------------------------------
tstp= -1 iter:  0 err ele: 164.653 err bos: 66.9483 - Local density matrix: (0.696125,0) and trace: (0.696125,0)
tstp= -1 iter:  1 err ele: 65.6358 err bos: 20.8254 - Local density matrix: (0.481072,0) and trace: (0.481072,0)
tstp= -1 iter:  2 err ele: 6.41737 err bos: 8.20878 - Local density matrix: (0.50032,0) and trace: (0.50032,0)
tstp= -1 iter:  3 err ele: 0.891083 err bos: 2.88073 - Local density matrix: (0.499776,0) and trace: (0.499776,0)
tstp= -1 iter:  4 err ele: 0.306784 err bos: 0.226723 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  5 err ele: 0.0312691 err bos: 0.150042 - Local density matrix: (0.499997,0) and trace: (0.499997,0)
tstp= -1 iter:  6 err ele: 0.0142294 err bos: 0.0115508 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  7 err ele: 0.00100715 err bos: 0.00691545 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  8 err ele: 0.00072276 err bos: 0.000556435 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  9 err ele: 9.78023e-05 err bos: 0.000331876 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  10 err ele: 3.26695e-05 err bos: 4.21535e-05 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  11 err ele: 4.71817e-06 err bos: 1.4927e-05 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  12 err ele: 1.47237e-06 err bos: 2.26511e-06 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  13 err ele: 3.12575e-07 err bos: 6.63213e-07 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  14 err ele: 6.34337e-08 err bos: 1.38834e-07 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  15 err ele: 1.59104e-08 err bos: 2.83693e-08 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  16 err ele: 2.59188e-09 err bos: 7.33601e-09 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  17 err ele: 9.01083e-10 err bos: 1.15735e-09 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  18 err ele: 1.01687e-10 err bos: 4.03234e-10 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  19 err ele: 4.54251e-11 err bos: 4.48629e-11 - Local density matrix: (0.5,0) and trace: (0.5,0)
..................................................
writing hdf5 data to out/k5.h5
Time [equilibrium calculation] = 5.67223s

writing hdf5 data to out/k0.h5
writing hdf5 data to out/k6.h5
writing hdf5 data to out/k1.h5
writing hdf5 data to out/k7.h5
writing hdf5 data to out/k8.h5
writing hdf5 data to out/k2.h5
writing hdf5 data to out/k9.h5
writing hdf5 data to out/k3.h5
writing hdf5 data to out/k4.h5


Time [total] = 5.96637s
************************************************************
```
