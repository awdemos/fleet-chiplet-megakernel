# AGENTS.md — fleet-chiplet-megakernel

## What this repo is

A research fork of the Mirage Persistent Kernel (MPK) framework that adds **Fleet**: cooperative, XCD-aware task scheduling for LLM inference megakernels on AMD MI350 multi-die GPUs. It keeps weights in per-XCD L2 cache and uses hierarchical synchronization to reduce cross-chiplet traffic.

Upstream: `ROCm/fleet-chiplet-megakernel` (branch `amd_mi350`). This fork tracks it.

## Repository layout

```
.
├── setup.py                 # Python package build entry point
├── pyproject.toml           # Minimal PEP 517 metadata
├── CMakeLists.txt           # C++/HIP build
├── config.cmake             # CMake option presets
├── README.md                # Paper results, setup, and Figure 6 reproduction
├── INSTALL.md               # Mirage upstream install instructions
├── CONTRIBUTING.md          # Contribution guidelines
├── NCU_Usage_Manual.md      # Nsight Compute profiling notes
├── analyze.sh               # Helper for profiling output
├── cpp_examples/            # Standalone C++ samples
├── demo/                    # LLM inference demos (Qwen3, etc.)
├── docker/                  # Docker build context
├── docker-build/            # Image build scripts
├── python/                  # Python bindings / runtime
├── src/                     # C++/HIP source (mirage + fleet runtime)
├── include/                 # Public headers
├── cmake/                   # CMake modules
├── scripts/                 # Utility scripts
├── tests/                   # Test suite
└── deps/                    # Git submodules (z3, etc.)
```

## Verified setup commands

### Clone and checkout the MI350 branch

```bash
git clone https://github.com/awdemos/fleet-chiplet-megakernel.git
cd fleet-chiplet-megakernel
git checkout amd_mi350
```

### Build the Python package from source

```bash
git submodule update --init --recursive
pip install -e . -v
export MIRAGE_HOME=$(pwd)
python -c "import mirage; print('Mirage OK')"
```

### Build the standalone C++ library

```bash
cd deps/z3 && mkdir build && cd build && cmake .. && make -j
cd ../..
export Z3_DIR=$PWD/deps/z3/build
mkdir build && cd build && cmake .. && make -j
```

### Run a quick smoke test

```bash
export MIRAGE_HOME=$(pwd)
python -c "import mirage"
```

## Reproducing the paper result (Figure 6)

Requires an AMD MI350, ROCm 7.0+, and the Qwen3-8B model.

```bash
export MIRAGE_HOME=$(pwd)
python3 -c "from huggingface_hub import snapshot_download; snapshot_download('Qwen/Qwen3-8B')"

rm -rf permanent_output_dir
HIP_VISIBLE_DEVICES=0 USE_GANG=1 USE_CK_FMHA=1 USE_FUSED_SILU=1 GANG_M_TILES_GATEUP=1 \
  python3 demo/qwen3/demo.py --use-mirage \
  --model Qwen/Qwen3-8B \
  --max-num-batched-tokens 1 --max-num-batched-requests 1 \
  --max-seq-length 1200 --max-new-tokens 1024 \
  --ignore-eos --prompt "Hello"
```

Delete `permanent_output_dir/` before switching modes.

## Code / contribution style

- C++ / HIP code uses `.clang-format` at the repo root; run `clang-format -i` on changed files.
- Python packaging is driven by `setup.py`; CMake handles the native build.
- Prefer environment-variable feature flags for kernel variants (e.g. `USE_GANG`, `USE_GANG_M_SPLIT`).
- Large generated kernel caches (`permanent_output_dir/`) are gitignored; do not commit them.

## Common gotchas

- **Submodules**: `setup.py` expects `deps/z3` and other submodules; run `git submodule update --init --recursive` first or the build will fail with missing headers.
- **MIRAGE_HOME**: many scripts expect this to point at the repo root; set it after building.
- **Kernel cache invalidation**: switching from `fleet` to `fleet_msplit` (or back) without deleting `permanent_output_dir/` will reuse a kernel compiled for the wrong mode.
- **MI350-only**: the Fleet scheduling logic assumes gfx950 / 8 XCD topology; it will not run on earlier AMD GPUs.
- **ROCm 7.0+ required**: hipcc and the runtime must match; older ROCm builds will fail on gfx950-specific code.

## Key files to read when changing something

- Build system: `CMakeLists.txt`, `config.cmake`, `setup.py`
- Scheduling/runtime: `src/`, `include/`
- Python demos: `demo/qwen3/demo.py`
- Kernel-variant flags: search for `USE_GANG`, `USE_GANG_M_SPLIT`, `GANG_M_TILES_GATEUP`
- Performance analysis: `analyze.sh`, `NCU_Usage_Manual.md`
