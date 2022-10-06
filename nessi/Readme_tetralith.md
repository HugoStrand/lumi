
# Building NESSi on Tetralith

## Module environment

```
ml buildtool-easybuild/4.6.0-nscd2783194f
ml foss/2018a
ml Eigen/3.3.4-nsc1
ml HDF5/1.10.1-nsc1
ml Python/3.6.7-nsc1
ml h5py/2.8.0-Python-3.6.7-HDF5-1.10.1-nsc1
ml CMake/3.23.2
ml git
```

```
[~]$ ml

Currently Loaded Modules:
  1) mpprun/4.3.0                                  10) hwloc/.1.11.8              (H)  19) GMP/6.1.2-nsc1
  2) nsc/.1.1                               (H,S)  11) OpenMPI/.2.1.2             (H)  20) libffi/3.2.1-nsc1
  3) EasyBuild/4.6.0-nscd2783194f                  12) foss/2018a                      21) Python/3.6.7-nsc1
  4) nsc-eb-scripts/1.3                            13) Eigen/3.3.4-nsc1                22) Szip/2.1.1-nsc1
  5) buildtool-easybuild/4.6.0-nscd2783194f        14) CMake/3.23.2                    23) HDF5/1.10.1-nsc1
  6) GCCcore/6.4.0                                 15) git/2.37.1-nsc1-gcc-system      24) pkg-config/0.29.2-nsc1
  7) binutils/.2.28                         (H)    16) ncurses/6.0-nsc1                25) pkgconfig/1.2.2-Python-3.6.7-nsc1
  8) GCC/6.4.0-2.28                                17) libreadline/7.0-nsc1            26) h5py/2.8.0-Python-3.6.7-HDF5-1.10.1-nsc1
  9) numactl/.2.0.11                        (H)    18) SQLite/3.21.0-nsc1

  Where:
   S:  Module is Sticky, requires --force to unload or purge
   H:             Hidden Module
```

## Clone NESSi

```bash
~/> cd dev
~/dev> git clone https://github.com/nessi-cntr/nessi.git
~/dev> cd nessi/libcntr
```


## Fix Eigen3 include path

```
~/dev/nessi/libcntr> mkdir eigen_include
~/dev/nessi/libcntr> ln -s /software/sse/easybuild/prefix/software/Eigen/3.3.4-nsc1/include eigen_include/eigen3
```

## Configure and build NESSi

Command line for `cmake` stored in the file `cmake.sh` (and made executable using `chmod +x cmake.sh`)
```
~/dev/nessi/libcntr> vi cmake.sh 
```

```
CC=gcc CXX=g++ \
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

```
~/dev/nessi/libcntr> chmod +x cmake.sh 
~/dev/nessi/libcntr> mkdir cbuild
~/dev/nessi/libcntr> cd cbuild/
~/dev/nessi/libcntr/cbuild> ../cmake.sh 
~/dev/nessi/libcntr/cbuild> make -j 16
~/dev/nessi/libcntr/cbuild> make install
```

## Configure and build NESSi example programs

```
~/dev/nessi/libcntr/cbuild> cd ../../examples
```

Disable automatic finding of the Eigen3 library
```
[examples]$ git diff
diff --git a/examples/CMakeLists.txt b/examples/CMakeLists.txt
index 193eded..979120b 100644
--- a/examples/CMakeLists.txt
+++ b/examples/CMakeLists.txt
@@ -44,8 +44,8 @@ if (mpi)
 endif ()
 
 # ~~ Add Eigen ~~
-find_package(Eigen3 REQUIRED)
-include_directories(SYSTEM ${EIGEN3_INCLUDE_DIR})
+#find_package(Eigen3 REQUIRED)
+#include_directories(SYSTEM ${EIGEN3_INCLUDE_DIR})
 
 # ~~ Paths and Subdirs ~~
 include_directories(${CMAKE_INCLUDE_PATH})
```

```
~/dev/nessi/examples> vi cmake.sh
```

```
CC=gcc CXX=g++ \
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

```
~/dev/nessi/examples> chmod +x cmake.sh 
~/dev/nessi/examples> mkdir cbuild
~/dev/nessi/examples> cd cbuild/
~/dev/nessi/examples/cbuild> ../cmake.sh
~/dev/nessi/examples/cbuild> make -j 16
~/dev/nessi/examples/cbuild> cd .. 
```

## Setup and run `demo_gw.py`

Add the NESSi python module and the NESSi library to your shell environment
```
export PYTHONPATH=$HOME/dev/nessi/libcntr/python3:$PYTHONPATH
export LD_LIBRARY_PATH=$HOME/apps/libcntr/lib:$LD_LIBRARY_PATH
```

Disable `matplotlib` in `demo_gw.py`
```
diff --git a/examples/utils/demo_gw.py b/examples/utils/demo_gw.py
index 77d1a65..f8d5a43 100644
--- a/examples/utils/demo_gw.py
+++ b/examples/utils/demo_gw.py
@@ -2,7 +2,7 @@ import sys
 import os
 import numpy as np
 import h5py
-import matplotlib.pyplot as plt
+#import matplotlib.pyplot as plt
 from ReadCNTR import write_input_file
 #----------------------------------------------------------------------
 def merge_two_dicts(x, y):
@@ -129,6 +129,7 @@ if __name__ == '__main__':
     fname = 'out/data_gw.h5'
     ts, Epulse, Ekin, Epot = ReadData(fname)
 
+    exit()
     fig, ax = plt.subplots(2,1,sharex=True)
     ax[0].plot(ts, Epulse, c='blue')
     ax[1].plot(ts, Ekin - Ekin[0], c='k')
```	 
	 
Run `demo_gw.py`
```bash
[examples]$ python utils/demo_gw.py 
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
tstp= -1 iter:  17 err ele: 9.01088e-10 err bos: 1.15735e-09 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  18 err ele: 1.01686e-10 err bos: 4.03236e-10 - Local density matrix: (0.5,0) and trace: (0.5,0)
tstp= -1 iter:  19 err ele: 4.54256e-11 err bos: 4.48618e-11 - Local density matrix: (0.5,0) and trace: (0.5,0)
..................................................
Time [equilibrium calculation] = 22.6861s

writing hdf5 data to out/k5.h5
writing hdf5 data to out/k6.h5
writing hdf5 data to out/k0.h5
writing hdf5 data to out/k7.h5
writing hdf5 data to out/k1.h5
writing hdf5 data to out/k8.h5
writing hdf5 data to out/k2.h5
writing hdf5 data to out/k9.h5
writing hdf5 data to out/k3.h5
writing hdf5 data to out/k4.h5


Time [total] = 22.7652s
************************************************************
```
