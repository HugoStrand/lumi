


```bash
./> git clone https://github.com/nessi-cntr/nessi.git
./> cd nessi/libcntr
```

Fix for relative include pattern of `Eigen3`
```
./nessi/libcntr> mkdir eigen_include
./nessi/libcntr> ln -s /appl/lumi/SW/LUMI-22.08/common/EB/Eigen/3.4/include eigen_include/eigen3
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
./nessi/libcntr> mkdir cbuild
./nessi/libcntr> cd cbuild
./nessi/libcntr/cbuild> ../cmake.sh
./nessi/libcntr/cbuild> make -j 128
./nessi/libcntr/cbuild> make install
```


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
./nessi/examples> mkdir cbuild
./nessi/examples/cbuild> ../cmake.sh
./nessi/examples/cbuild> make -j 128
./nessi/examples/cbuild> cd ..
```

```
export PYTHONPATH=$HOME/dev/nessi/libcntr/python3:$PYTHONPATH
export LD_LIBRARY_PATH=$HOME/apps/libcntr/lib:$LD_LIBRARY_PATH
```

Change MPI run command from `mpirun` to `srun` in `./nessi/examples/utils/demo_gw.py`
```bash
sed -i bak 's/mpirun/srun/g' ./utils/demo_gw.py
git diff 
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

```bash
./nessi/examples> salloc --nodes=1 --ntasks=128 --partition=debug --account=project_465000175 --time=00:05:00
salloc: Pending job allocation 1748084
salloc: job 1748084 queued and waiting for resources
salloc: job 1748084 has been allocated resources
salloc: Granted job allocation 1748084
./nessi/examples> python ./utils/demo_gw.py
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
