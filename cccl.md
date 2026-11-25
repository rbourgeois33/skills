# general: Applies to work on CCCL.

- I am in **learning mode** on CCCL: always explain how things were chosen/run (which existing benchmark/test, why, which code path), and say what is existing code vs. what you wrote.
- Upstream: https://github.com/NVIDIA/cccl. Read the repo's `AGENTS.md`/`CLAUDE.md` first.

## Current work (2026-11)
- NVIDIA/cccl#3115 (EPIC: programmatic dependent launch (PDL) in all CUB algorithms). My target: `cub::DeviceReduce`, **run_to_run** path (2-kernel reduction: hide 2nd kernel launch latency). Maintainer (bernhardmgruber): "please supply proper benchmarks for any PR".

## Layout on Jean Zay
- Source: `$WORK/cccl` (fresh clone of upstream main; baseline commit `8738cade`, 2026-11-25).
- Jobs/logs/results: `$WORK/cccl-work/{jobs,logs,results}`; CPM cache: `$WORK/cccl-work/cpm-cache`.
- Build trees: `/lustre/fsn1/projects/rech/pri/ujq29su/cccl-build/<branch>-<sha>-cub-benchmark-h100`.
- Watcher: `$WORK/cccl-work/jobs/watch.sh JOBID LOG` (returns when job leaves queue or log shows `error:`/`FAILED:`; run in background).

## Build benchmarks (H100)
- Configure + build in ONE job on `--partition=compil_h100` (has internet, so CPM fetches nvbench etc.; no GPU there). Script: `$WORK/cccl-work/jobs/build_h100.sh`.
  - modules: `arch/h100 gcc/13.3.0 cuda/12.8.0 cmake/3.31.4 ninja/1.12.1 git/2.39.1` (git module needed by CPM).
  - `cmake --preset cub-benchmark -B <build> -DCMAKE_CUDA_ARCHITECTURES=90 -DCPM_SOURCE_CACHE=$WORK/cccl-work/cpm-cache`
    (preset default arch is `native`, which fails on a GPU-less compil node -> force 90).
  - `cmake --build <build> --target cub.bench.reduce.sum.base -j 24` -> ~30 s configure + ~1 min build. Build only the targets you need.
- Benchmark target names: `cub.bench.<algo>.<file>.base` from `cub/benchmarks/bench/<algo>/<file>.cu` (`.base` = TUNE_BASE, default policies).

## Run benchmarks
- **Always `--exclusive`** (whole node, billed 4 GPUs): `-A trt@h100 -C h100 --qos=qos_gpu_h100-dev --gres=gpu:1 --exclusive`. Script: `$WORK/cccl-work/jobs/run_reduce_sum_h100.sh`.
- `bin/<bench> -d 0 --json out.json --md out.md`; also save `nvidia-smi -q -d CLOCK,PERFORMANCE,COMPUTE`.
- Compare two runs with nvbench's `compare.py` (in the CPM nvbench source under `scripts/`).

## DeviceReduce benchmark map (`cub/benchmarks/bench/reduce/`)
- `sum.cu` (+ `base.cuh`): `DeviceReduce::Reduce(..., env)` with no determinism requirement -> default = **run_to_run** (`cub/cub/device/device_reduce.cuh` ~l.160, 325). This is the run_to_run benchmark. Axes: all types x {I32,I64} offsets x 2^16,20,24,28.
- `min.cu`, `custom.cu`: same base, other ops (also run_to_run).
- `deterministic.cu`: requires **gpu_to_gpu** (not run_to_run despite the name). `nondeterministic.cu`: atomics path.
- Baseline H100 (8738cade): F32/I32 GPU time 9.1 us @2^16, 11.3 us @2^20, 38.4 us @2^24, 361 us @2^28 (88.6% BW). CPU time ~37 us at small sizes. Results: `$WORK/cccl-work/results/baseline-8738cade/`.
