# JeanZay: All tips to work smoothly on the jeanzay supercomputer

Add here everyhting you need to know about my account or about how jeanzay works here.

- doc: https://www.idris.fr/docs/jean-zay/jean-zay/jean-zay-presentation/

## My account
- Be mindful about my hours consumption. I only have 4000/year. Compile on compile node when you can. Use exclusive ONLY if you are doing benchmarks.
- Unix groups: `pri`, `trt`. GPU hours for TRUST work: `-A trt@h100 -C h100`. One GPU is enough for build+tests: `--gres=gpu:1` (H100 node = 4 GPUs, 96 cores / 192 threads).
- Work dir: `/lustre/fswork/projects/rech/pri/ujq29su`; TRUST tmp/test dir: `/lustre/fsn1/projects/rech/pri/ujq29su/.tmp_TRUST_ujq29su`.

## Rules
- Never build on the login node (frontale). Build/test only through `sbatch`. Never game Slurm; waiting in the queue is fine.
- Downloads (git lfs pull, pip, git clone, ...) must be done on the login node: GPU compute nodes have **no internet**.
- Exception: `--partition=compil` / `compil_h100` nodes have internet (proxy): CMake configures that download deps (CPM/FetchContent) go there (no GPU, so set CUDA arch explicitly).
- Benchmarks: always `--exclusive` (whole node; `scontrol show job` then shows AllocTRES gres/gpu=4, billing=4).

## How the machine behaves
- Hostnames: login `jean-zay*`, H100 compute nodes `jzxh*` (e.g. `jzxh202`).
- Compute nodes have fewer system packages than login: no `swig` (login has `/usr/bin/swig`), no python3 headers in `/usr/include`.
- Module `python/3.11.5` = anaconda env `/lustre/fshomisc/sup/pub/anaconda-py3/2024.06/envs/python-3.11.5` (has libpython + headers), but CMake's legacy `FindPythonLibs` cannot find it (on Linux it doesn't derive `<prefix>/lib` from PATH, module sets no `CMAKE_PREFIX_PATH`).
- Sourcing envs/modules in a batch job prints `conda.sh: No such file or directory` / `conda: command not found` and returns non-zero: don't use `set -e` around `source`.
- `error: Unable to create TMPDIR []` at job start is harmless.
- Spack OpenMPI (`openmpi/4.1.5-cuda`, `4.1.8-cuda`): `mpirun`/`mpiexec` are stubs ("installed without the mpiexec/mpirun commands ... use srun"). Launch MPI with `srun` only. To run several MPI steps concurrently inside one allocation: `export SLURM_OVERLAP=1` and allocate enough tasks (`--ntasks=24 --cpus-per-task=1`).
- The Slurm controller is sometimes very slow: `squeue`/`scancel` can hang > 60 s. Wrap them in `timeout 100`.
- Cancelled jobs stay `COMPLETING` in `squeue` for a while.
- Pending reasons seen: `Priority`, `QOSGrpGRES` (group GPU quota), `QOSMaxCpuPerJobLimit`.
