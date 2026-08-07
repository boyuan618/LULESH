# Building and Running LULESH on ASPIRE (Cray EX)

LULESH 2.0 is LLNL's Livermore Unstructured Lagrangian Explicit Shock
Hydrodynamics proxy app. It builds with MPI + OpenMP and takes ~30 seconds
to compile. Total time for this guide: about 5 minutes plus queue wait.

## 1. Get the source

```bash
cd /scratch/users/ntu/<user>/workshop
git clone https://github.com/llnl/LULESH.git
cd LULESH
```

## 2. Set up the environment

The system loads `PrgEnv-cray` by default. Swap to the GNU programming
environment and load CMake — the Cray compiler wrappers `cc`/`CC` then wrap
GCC and link `cray-mpich` automatically (no `mpicxx` needed):

```bash
module swap PrgEnv-cray PrgEnv-gnu
module load cmake/3.31.3
```

## 3. Configure and build

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER=CC ..
make -j8
```

You should see CMake report `Found MPI: TRUE` and `Found OpenMP: TRUE`
(both come through the `CC` wrapper), and the build produces `lulesh2.0`.

## 4. Run via PBS

Do not run the binary on the login node — cray-mpich binaries need a proper
launcher and will abort. Submit a job to the `normal` queue instead.

**Important:**
- LULESH requires the number of MPI ranks to be a perfect cube
  (1, 8, 27, 64, ...).
- Small jobs on the `normal` queue route to `qdev`, which caps the queue's
  *total* footprint at 256 CPUs across all users' jobs. Keep workshop test
  jobs small (e.g. 8 CPUs) or a large request can sit in `Q` indefinitely
  with the comment "would exceed overall limit on resource ncpus".

Save as `run_lulesh.pbs` in the LULESH directory:

```bash
#!/bin/bash
#PBS -N lulesh_test
#PBS -P <project-id>            # e.g. personal-<user>
#PBS -q normal
#PBS -l select=1:ncpus=8:mpiprocs=8:ompthreads=1:mem=16gb
#PBS -l walltime=00:10:00
#PBS -j oe
#PBS -o lulesh_test.log

module swap PrgEnv-cray PrgEnv-gnu 2>/dev/null

cd /scratch/users/ntu/<user>/workshop/LULESH/build
export OMP_NUM_THREADS=1

mpirun -np 8 ./lulesh2.0 -s 30 -i 100 -p
```

Submit and watch:

```bash
qsub run_lulesh.pbs
qstat -u $USER
```

## 5. Command-line options

| Flag | Meaning |
|------|---------|
| `-s N` | Problem size: N^3 elements per MPI rank (default 30) |
| `-i N` | Max number of iterations (default: run to completion) |
| `-p` | Print progress every cycle |
| `-r N` | Number of regions (default 11) |
| `-q` | Quiet mode |

## 6. Expected output

The job prints one line per cycle, then a summary. From an actual run of the
script above (8 ranks, `-s 30 -i 100`, ~3 s of compute):

```
Run completed:
   Problem size        =  30
   MPI tasks           =  8
   Iteration count     =  100
   Final Origin Energy =  1.058138e+07
   Testing Plane 0 of Energy Array on rank 0:
        MaxAbsDiff   = 5.820766e-10
        TotalAbsDiff = 1.593827e-09
        MaxRelDiff   = 2.883318e-13

Elapsed time         =        3.1 (s)
Grind time (us/z/c)  =  1.1586444 (per dom)  (   3.12834 overall)
FOM                  =  6904.6204 (z/s)
```

The `MaxRelDiff` around 1e-13 is the built-in symmetry check — values that
small mean the run is numerically correct.

## Notes

- Scaling up: keep ranks a cube. E.g. 27 ranks
  (`ncpus=27:mpiprocs=27`) or 64 ranks (`ncpus=64:mpiprocs=64`). Anything
  beyond a few dozen CPUs will contend with the 256-CPU `qdev` cap — for
  real scaling runs use a project with access to the large-job queues.
- Hybrid MPI+OpenMP also works: set `ompthreads`/`OMP_NUM_THREADS` > 1 and
  add `--depth <threads>` to `mpirun` so ranks are spaced across cores.
- `-s` is the per-rank problem size, so total elements = ranks × s³
  (weak scaling by default).
- The figure of merit is the final `FOM` line (zone-cycles per second).
