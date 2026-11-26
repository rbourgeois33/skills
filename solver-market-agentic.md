# solver-market-agentic: tooling for an agent that picks and tunes GPU AMG (MueLu / BoomerAMG / GAMG)

- Repo: `git@github.com:rbourgeois33/solver-market-agentic.git`, clone at `$WORK/solver-market-agentic`.
  Design agreed with me: `docs/design.md` (read it first). Work branch `phase1`.
- State dir `SMA_WORK=$WORK/sma-work`: `spack/` (v1.2.2), `spack-user/` (spack config+cache, NOT in HOME),
  `env/` (rendered spack env + `build_log.md`), `install/`, `view/`, `bin/` (drivers), `data/` (converted
  systems), `logs/` (job outputs + `watch.sh`). Build stage: `$SCRATCH/sma-stage`.

## Decisions (2026-11-26, all asked to me)
- GPU only, portable libs: Trilinos 17.3.0 (Tpetra + MueLu + Belos, `+cusparse`; **Epetra is deprecated,
  never use it**), hypre 3.2.1, PETSc 3.26.1 (+kokkos). Kokkos/KK 5.2.2. Pin true upstream latest.
- Toolchain on Jean Zay H100: `arch/h100 gcc/13.3.0 cuda/12.8.0 openmpi/4.1.6-cuda cmake/3.31.4`.
- Each lib uses its own Krylov. Decks = YAML with the library's own parameter names; only keys present are set.
- Objective = runtime: cost = (t_setup + k t_solve)/k, k = solves per setup, k=0 -> t_solve only.
  Convergence = TRUE relative residual recomputed by the driver. Memory = peak GPU mem (constraint only).
- Persistent driver servers: matrix read + uploaded once, many configs. Own binary CSR format.
- Cluster: orchestrator + agent on login, worker = sbatch on exclusive H100 node fed by a Lustre queue.
  The worker must die when the agent stops (STOP file + scancel + idle timeout + walltime).
- Agents via CLI adapters (`claude -p` or any CLI agent). Budget enforced by `sma-tools run` (needs `--why`).
- Expert decks live in `bench/` of the same repo (public info, we trust the agent). Matrices: public HF dataset.
- The agent may edit the spack env, but must log every tweak in `$SMA_WORK/env/build_log.md`.

## Spack on Jean Zay (learned the hard way)
- BUILD the GPU libs ON AN H100 NODE (`machines/jean-zay/build_libs_gpu.slurm`): configure test programs linking CUDA
  cannot run on compil_h100 (no libcuda) -> PETSc silently assumed 64-bit BLAS indices -> GAMG crash in LAPACKgeqrf.
  compil_h100 only for concretize + source download (GPU nodes are offline: sources must be in the spack cache).
- spack-built OpenBLAS 0.3.33 .so had a misaligned LOAD segment (glibc refuses it): BLAS = netlib-lapack.
- Installed drivers need RPATH (`CMAKE_INSTALL_RPATH_USE_LINK_PATH=ON`), and a clean CMake build dir after any env
  rebuild (cached HYPRE_DIR/Trilinos_DIR point to old hashes). Linking Kokkos on a GPU-less node needs the libcuda stub.
- Never concretize/build on login: target differs (login = cascadelake, compil_h100/H100 nodes = sapphirerapids) and module paths may differ.
- Spack does NOT expand env variables in external `prefix`/`compilers` paths: `sma-tools spack-env`
  writes real paths (from the loaded modules) into `$SMA_WORK/env/packages.yaml`.
- A manually declared gcc external needs `extra_attributes: compilers: {c, cxx, fortran}` (else `SPACK_CC=None`,
  "C compiler cannot create executables").
- `SPACK_PYTHON=/usr/bin/python3`: the `python/3.11.5` module (arch/h100) lacks `os.O_PATH` -> spack 1.2 crashes.
- Trilinos 17 removed Epetra/AztecOO/ML/Ifpack/Amesos: naming `~epetra` etc. silently pins trilinos@:16.
- builtin petsc recipe requires `kokkos+cuda_lambda`, which no longer exists in Kokkos 5: overlay fixes it.
- `build_libs.slurm` must stop if concretize fails (otherwise `spack install` reuses an old spack.lock).

## Testing conventions
- Never run analysis/conversion of real matrices on the login node (the `--eigs` Lanczos on a 52k matrix took 9 min).
- Test matrices: `$WORK/aij_51840.mtx` (dev), `$WORK/aij_2592000.mtx` (first real target), rhs = ones.
  aij_51840: SPD-like, M-matrix pattern, 98.9% zero row sums, cond(D^-1/2 A D^-1/2) ~ 2.8e4 -> CG + AMG.
- Job watcher: `$SMA_WORK/logs/watch.sh JOBID LOG` (sacct state + grep `==> Error` / `[x] ... failed`); run in background.

## Results so far (aij_2592000, rhs=1, tol 1e-8 true residual, 1 H100, solve time only)
- hypre PCG+BoomerAMG GPU defaults = PMIS/ext+i/l1-Jacobi deck: 22 its, 46.6 ms solve, 41 ms setup, op cx 3.46.
- MueLu SA defaults: 4.15 s solve (default smoother bad on GPU); SA + Chebyshev(2): 17 its, 27.5 ms solve.
