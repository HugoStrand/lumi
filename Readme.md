# Lumi notes

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

## Extra packages for Triqs

Need to install some python modules by hand

```bash
pip install --user h5py
pip install --user mako
```

## Triqs CMake command

Generate make files using `cmake`
```bash
CXX=CC CC=cc cmake \
  -DCMAKE_INSTALL_PREFIX=$HOME/apps/triqs \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-static-libstdc++ -O3 -march=znver2 -mtune=znver2 -mfma -mavx2 -m3dnow -fomit-frame-pointer" \
  ..
```
(Todo: Tune the optimization parameters for Lumi zen3 cpus above.)

Compile in parallell
```bash
make -j 128
```

Setup environment variables for Triqs using:
```
source /users/hustrand/apps/triqs/share/triqs/triqsvars.sh
```

## Triqs test

The Triqs test suite can be run from the login node by starting an interactive jobsession
```bash
salloc --nodes=1 --partition=standard --account=project_465000175 --time=00:30:00
ctest -j 64
```

Note that the tests can not be executed in a shell on the compute note
```
## srun --cpu_bind=none --nodes=1 --pty bash -i ## DO NOT START SHELL ON NODE
```
since `Triqs` runs `srun` internally for the mpi based tests.

With this setup the majority of tests passes. There are two test failing currently:
```bash
97% tests passed, 7 tests failed out of 248

Total Test time (real) =  21.93 sec

The following tests FAILED:
	  8 - mpi_custom_np2 (Failed)
	  9 - mpi_custom_np4 (Failed)
	 10 - mpi_monitor_np2 (Failed)
	 11 - mpi_monitor_np4 (Failed)
	172 - different_moves_mc_np2 (Failed)
	173 - different_moves_mc_np3 (Failed)
	174 - different_moves_mc_np4 (Failed)
```	

```
hustrand@uan03:~/dev/triqs/build> ctest -V -R mpi_custom_np2
UpdateCTestConfiguration  from :/users/hustrand/dev/triqs/build/DartConfiguration.tcl
UpdateCTestConfiguration  from :/users/hustrand/dev/triqs/build/DartConfiguration.tcl
Test project /users/hustrand/dev/triqs/build
Constructing a list of tests
Done constructing a list of tests
Updating test list for fixtures
Added 0 tests to meet fixture requirements
Checking test dependency graph...
Checking test dependency graph end
test 8
    Start 8: mpi_custom_np2

8: Test command: /usr/bin/srun "-n" "4" "/users/hustrand/dev/triqs/build/deps/mpi/test/c++//mpi_custom"
8: Test timeout computed to be: 10000000
8: [==========] Running 3 tests from 1 test suite.
8: [----------] Global test environment set-up.
8: [----------] 3 tests from MPI_CUSTOM
8: [ RUN      ] MPI_CUSTOM.custom_type_op
8: [==========] Running 3 tests from 1 test suite.
8: [----------] Global test environment set-up.
8: [----------] 3 tests from MPI_CUSTOM
8: [ RUN      ] MPI_CUSTOM.custom_type_op
8: [==========] Running 3 tests from 1 test suite.
8: [----------] Global test environment set-up.
8: [----------] 3 tests from MPI_CUSTOM
8: [ RUN      ] MPI_CUSTOM.custom_type_op
8: [==========] Running 3 tests from 1 test suite.
8: [----------] Global test environment set-up.
8: [----------] 3 tests from MPI_CUSTOM
8: [ RUN      ] MPI_CUSTOM.custom_type_op
8: Attempting to use an MPI routine before initializing MPICH
8: Attempting to use an MPI routine before initializing MPICH
8: Attempting to use an MPI routine before initializing MPICH
8: Attempting to use an MPI routine before initializing MPICH
8: srun: error: nid001394: tasks 0-3: Exited with exit code 1
8: srun: launch/slurm: _step_signal: Terminating StepId=1438131.76
1/1 Test #8: mpi_custom_np2 ...................***Failed    0.33 sec

0% tests passed, 1 tests failed out of 1

Total Test time (real) =   0.37 sec

The following tests FAILED:
	  8 - mpi_custom_np2 (Failed)
Errors while running CTest
```

```
hustrand@uan03:~/dev/triqs/build> ctest -V -R different_moves_mc_np2
UpdateCTestConfiguration  from :/users/hustrand/dev/triqs/build/DartConfiguration.tcl
UpdateCTestConfiguration  from :/users/hustrand/dev/triqs/build/DartConfiguration.tcl
Test project /users/hustrand/dev/triqs/build
Constructing a list of tests
Done constructing a list of tests
Updating test list for fixtures
Added 0 tests to meet fixture requirements
Checking test dependency graph...
Checking test dependency graph end
test 172
    Start 172: different_moves_mc_np2

172: Test command: /usr/bin/srun "-n" "2" "/users/hustrand/dev/triqs/build/test/c++/mc_tools/different_moves_mc"
172: Test timeout computed to be: 10000000
172: [==========] Running 2 tests from 1 test suite.
172: [----------] Global test environment set-up.
172: [----------] 2 tests from mc_generic
172: [ RUN      ] mc_generic.WithException
172: [==========] Running 2 tests from 1 test suite.
172: [----------] Global test environment set-up.
172: [----------] 2 tests from mc_generic
172: [ RUN      ] mc_generic.WithException
172: HDF5-DIAG: Error detected in HDF5 (1.12.1) thread 0:
172:   #000: ../../src/H5F.c line 532 in H5Fcreate(): unable to create file
172:     major: File accessibility
172:     minor: Unable to open file
172:   #001: ../../src/H5VLcallback.c line 3282 in H5VL_file_create(): file create failed
172:     major: Virtual Object Layer
172:     minor: Unable to create file
172:   #002: ../../src/H5VLcallback.c line 3248 in H5VL__file_create(): file create failed
172:     major: Virtual Object Layer
172:     minor: Unable to create file
172:   #003: ../../src/H5VLnative_file.c line 63 in H5VL__native_file_create(): unable to create file
172:     major: File accessibility
172:     minor: Unable to open file
172:   #004: ../../src/H5Fint.c line 1898 in H5F_open(): unable to lock the file; consider setting HDF5_USE_FILE_LOCKING
172:     major: File accessibility
172:     minor: Unable to lock file
172:   #005: ../../src/H5FD.c line 1625 in H5FD_lock(): driver lock request failed
172:     major: Virtual File Layer
172:     minor: Unable to lock file
172:   #006: ../../src/H5FDsec2.c line 1002 in H5FD__sec2_lock(): unable to lock file, errno = 11, error message = 'Resource temporarily unavailable'
172:     major: Virtual File Layer
172:     minor: Unable to lock file
172: unknown file: Failure
172: C++ exception with description "HDF5 : cannot createfile : params.h5" thrown in the test body.
172: [  FAILED  ] mc_generic.WithException (2 ms)
172: [ RUN      ] mc_generic.WithOutException
172: 
172: Warming up ...
172: 
172: Accumulating ...
172: 
172: Warming up ...
172: 
172: Accumulating ...
172: 18:34:30  33% ETA 00:00:00 cycle 33326 of 100000
172: 18:34:30  33% ETA 00:00:00 cycle 33218 of 100000
172: 
172: 
172: 
172: 
172: HDF5-DIAG: Error detected in HDF5 (1.12.1) thread 0:
172:   #000: ../../src/H5F.c line 532 in H5Fcreate(): unable to create file
172:     major: File accessibility
172:     minor: Unable to open file
172:   #001: ../../src/H5VLcallback.c line 3282 in H5VL_file_create(): file create failed
172:     major: Virtual Object Layer
172:     minor: Unable to create file
172:   #002: ../../src/H5VLcallback.c line 3248 in H5VL__file_create(): file create failed
172:     major: Virtual Object Layer
172:     minor: Unable to create file
172:   #003: ../../src/H5VLnative_file.c line 63 in H5VL__native_file_create(): unable to create file
172:     major: File accessibility
172:     minor: Unable to open file
172:   #004: ../../src/H5Fint.c line 1898 in H5F_open(): unable to lock the file; consider setting HDF5_USE_FILE_LOCKING
172:     major: File accessibility
172:     minor: Unable to lock file
172:   #005: ../../src/H5FD.c line 1625 in H5FD_lock(): driver lock request failed
172:     major: Virtual File Layer
172:     minor: Unable to lock file
172:   #006: ../../src/H5FDsec2.c line 1002 in H5FD__sec2_lock(): unable to lock file, errno = 11, error message = 'Resource temporarily unavailable'
172:     major: Virtual File Layer
172:     minor: Unable to lock file
172: [Rank 0] Collect results: Waiting for all mpi-threads to finish accumulating...
172: TEST : Exception occurred. Node 0 is stopping cleanly
172: /users/hustrand/dev/triqs/test/c++/mc_tools/different_moves_mc.cpp:183: Failure
172: Value of: stopped_by_exception
172:   Actual: true
172: Expected: false
172: [  FAILED  ] mc_generic.WithOutException (302 ms)
172: [----------] 2 tests from mc_generic (305 ms total)
172: 
172: [----------] Global test environment tear-down
172: [==========] 2 tests from 1 test suite ran. (305 ms total)
172: [  PASSED  ] 0 tests.
172: [  FAILED  ] 2 tests, listed below:
172: [  FAILED  ] mc_generic.WithException
172: [  FAILED  ] mc_generic.WithOutException
172: 
172:  2 FAILED TESTS
172: [Rank 0] Collect results: Waiting for all mpi-threads to finish accumulating...
172: [Rank 0] Timings for all measures:
172: Measure            | seconds   
172: histogramn measure | 0.00595847
172: Total measure time | 0.00595847
172: [Rank 0] Acceptance rate for all moves:
172: Move  left move: 0.399877
172: Move  right move: 1
172: [Rank 0] Warmup lasted: 0 seconds [00:00:00]
172: [Rank 0] Simulation lasted: 0.299194 seconds [00:00:00]
172: [Rank 0] Number of measures: 100000
172: Total number of measures: 100000
172: [       OK ] mc_generic.WithException (305 ms)
172: [ RUN      ] mc_generic.WithOutException
172: 
172: Warming up ...
172: 
172: Accumulating ...
172: 18:34:30  34% ETA 00:00:00 cycle 34253 of 100000
172: srun: error: nid001394: task 0: Exited with exit code 1
172: srun: launch/slurm: _step_signal: Terminating StepId=1438131.75
172: slurmstepd: error: *** STEP 1438131.75 ON nid001394 CANCELLED AT 2022-08-22T18:34:30 ***
172: TRIQS : Received signal 15
172: mc_generic stops because of a signal
172: 
172: 
172: mc_generic: Signal caught on node 0
172: 
172: [Rank 0] Collect results: Waiting for all mpi-threads to finish accumulating...
172: [Rank 0] Timings for all measures:
172: Measure            | seconds   
172: histogramn measure | 0.004318  
172: Total measure time | 0.004318  
172: [Rank 0] Acceptance rate for all moves:
172: Move  left move: 0.399583
172: Move  right move: 1
172: [Rank 0] Warmup lasted: 0 seconds [00:00:00]
172: [Rank 0] Simulation lasted: 0.10886 seconds [00:00:00]
172: [Rank 0] Number of measures: 37304
172: Total number of measures: 37304
172: [       OK ] mc_generic.WithOutException (115 ms)
172: [----------] 2 tests from mc_generic (420 ms total)
172: 
172: [----------] Global test environment tear-down
172: [==========] 2 tests from 1 test suite ran. (420 ms total)
172: [  PASSED  ] 2 tests.
1/1 Test #172: different_moves_mc_np2 ...........***Failed    0.69 sec

0% tests passed, 1 tests failed out of 1

Total Test time (real) =   0.72 sec

The following tests FAILED:
	172 - different_moves_mc_np2 (Failed)
Errors while running CTest
```
