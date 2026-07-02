# Building PyTorch on Windows — Matching Official CI Configs

This document explains how to reproduce the exact build configurations used by the
PyTorch maintainers for the official Windows **x64** and **ARM64** wheel releases,
as reported by `torch.__config__.show()`.

<!-- toc -->

- [Config Comparison: x64 vs ARM64](#config-comparison-x64-vs-arm64)
- [Windows x64 Build](#windows-x64-build)
  - [x64 Prerequisites](#x64-prerequisites)
  - [x64 Build Steps](#x64-build-steps)
- [Windows ARM64 Build](#windows-arm64-build)
  - [ARM64 Prerequisites](#arm64-prerequisites)
  - [ARM64 Build Steps](#arm64-build-steps)
- [Optional: sccache for Faster Rebuilds](#optional-sccache-for-faster-rebuilds)
- [Verifying Your Build](#verifying-your-build)

<!-- tocstop -->

---

## Config Comparison: x64 vs ARM64

| Feature | x64 | ARM64 |
|---|---|---|
| `BLAS` | MKL (`BLAS_INFO=mkl`) | APL (`BLAS_INFO=apl`) |
| `LAPACK` | MKL (`LAPACK_INFO=mkl`) | APL (`LAPACK_INFO=apl`) |
| `USE_MKLDNN` | ON | OFF |
| `USE_XNNPACK` | ON | OFF |
| `USE_PTHREADPOOL` | ON | OFF |
| `USE_MKL` | ON | OFF |
| CPU capability | AVX2 | DEFAULT |
| `USE_GLOO` | ON | ON |
| `USE_OPENMP` | ON | ON |
| `USE_CUDA` | OFF | OFF |
| `USE_KINETO` | ON | ON |
| CXX compiler | sccache-cl.exe (x64) | `Hostarm64/arm64/cl.exe` |
| `TORCH_VERSION` | 2.12.1 | 2.12.0 |

> **Note on `CPU capability usage: DEFAULT` (ARM64):** This is expected.
> PyTorch's CPU dispatch system on ARM64 does not use x86 AVX2/AVX512 paths.
> The `DEFAULT` capability is set automatically — no special flag is needed.

---

## Windows x64 Build

### x64 Prerequisites

1. **Visual Studio 2022** with the *Desktop development with C++* workload
   - Download: https://visualstudio.microsoft.com/vs/
   - The CI uses the Enterprise edition; Community edition works the same way
2. **Python 3.10+** — x64 installer
   - Download: https://www.python.org/downloads/windows/
3. **Intel oneAPI Base Toolkit** — provides MKL 2026.0 (`BLAS_INFO=mkl`)
   - Download: https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit-download.html
   - Only the *Intel oneAPI Math Kernel Library* component is strictly required
4. **CMake 3.18+** and **Ninja** — install via pip after creating your venv:
   ```bat
   pip install cmake ninja
   ```
5. **Git** with submodule support
6. At least **10 GB** of free disk space and **30–60 minutes** for the initial build

### x64 Build Steps

Save the following as `build_x64.bat` in the root of the cloned repo, or run
each block manually in a plain `cmd.exe` window (not PowerShell).

```bat
@echo on

:: ── 1. Clone source ───────────────────────────────────────────────────────────
git clone https://github.com/pytorch/pytorch.git
cd pytorch
git checkout v2.12.1
git submodule sync
git submodule update --init --recursive

:: ── 2. Activate Visual Studio x64 toolchain ──────────────────────────────────
::    Adjust the path if you have Professional or Community edition
call "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat" x64

:: ── 3. Activate Intel oneAPI (MKL) ───────────────────────────────────────────
call "C:\Program Files (x86)\Intel\oneAPI\setvars.bat"

:: ── 4. Create venv and install Python dependencies ───────────────────────────
python -m venv .venv
call .\.venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install cmake ninja

:: ── 5. Set build environment variables ───────────────────────────────────────

set CMAKE_BUILD_TYPE=Release
set BUILD_TYPE=Release

:: BLAS = MKL  (matches BLAS_INFO=mkl, LAPACK_INFO=mkl)
set BLAS=MKL
set USE_MKL=ON
set USE_LAPACK=1

:: oneDNN / MKL-DNN ON  (matches USE_MKLDNN=ON)
set USE_MKLDNN=ON

:: XNNPACK + pthreadpool ON  (matches USE_XNNPACK=ON, -DUSE_PTHREADPOOL in CXX_FLAGS)
set USE_XNNPACK=ON
set USE_PTHREADPOOL=ON

:: Threading
set USE_OPENMP=ON

:: Distributed
set USE_GLOO=ON
set USE_MPI=OFF
set USE_NCCL=OFF

:: No GPU
set USE_CUDA=0
set USE_CUDNN=OFF
set USE_ROCM=OFF
set USE_XPU=OFF
set USE_XCCL=OFF

:: Profiling — Kineto without CUPTI / ROCTracer
set USE_KINETO=ON
set LIBKINETO_NOCUPTI=ON
set LIBKINETO_NOROCTRACER=ON
set LIBKINETO_NOXPUPTI=ON

:: Misc
set USE_NNPACK=OFF
set USE_GFLAGS=OFF
set USE_GLOG=OFF
set INSTALL_TEST=0
set BUILD_TEST=0

:: Version  (matches TORCH_VERSION=2.12.1)
set PYTORCH_BUILD_VERSION=2.12.1
set PYTORCH_BUILD_NUMBER=1

:: ── 6. Build wheel ────────────────────────────────────────────────────────────
python -m build --wheel --no-isolation --outdir dist\

:: ── 7. Install ────────────────────────────────────────────────────────────────
pip install dist\torch-2.12.1-*.whl
```

---

## Windows ARM64 Build

### ARM64 Prerequisites

1. **Windows on ARM64 machine** — e.g., a device with Snapdragon X Elite / X Plus,
   or any other Windows-on-ARM hardware
2. **Visual Studio 2022** with the *ARM64 native build tools* component:
   - Component name: `MSVC v143 - VS 2022 C++ ARM64 build tools`
   - This gives you `Hostarm64/arm64/cl.exe`, matching the CI's `CXX_COMPILER`
3. **Python 3.10+** — ARM64 native installer
   - Download: https://www.python.org/downloads/windows/ (select the ARM64 build)
4. **Arm Performance Libraries (APL)** — provides BLAS + LAPACK (`BLAS_INFO=apl`)
   - Download: https://developer.arm.com/downloads/-/arm-performance-libraries
   - Note the install path; you will set `APL_ROOT` to it below
5. **libuv** built for ARM64 — the CI explicitly copies `uv.dll` into `torch\lib\`
   - Option A: Build from source targeting ARM64:
     https://github.com/libuv/libuv
   - Option B: `conda install -c conda-forge libuv` on an ARM64 conda environment
6. **CMake 3.18+** and **Ninja** — install via pip after creating your venv:
   ```bat
   pip install cmake ninja
   ```
7. **Git** with submodule support
8. At least **10 GB** of free disk space and **30–60 minutes** for the initial build

### ARM64 Build Steps

Save the following as `build_arm64.bat` in the root of the cloned repo, or run
each block manually in a plain `cmd.exe` window (not PowerShell).

```bat
@echo on

:: ── 1. Clone source ───────────────────────────────────────────────────────────
git clone https://github.com/pytorch/pytorch.git
cd pytorch
git checkout v2.12.0
git submodule sync
git submodule update --init --recursive

:: ── 2. Activate Visual Studio ARM64 native toolchain ─────────────────────────
::    "arm64" here selects Hostarm64/arm64/cl.exe — matching the CI CXX_COMPILER
call "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvarsall.bat" arm64

:: Confirm the right compiler is on PATH
where cl.exe

:: ── 3. Create venv and install Python dependencies ───────────────────────────
python -m venv .venv
call .\.venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install cmake ninja

:: ── 4. Copy libuv dll into torch\lib\ (as the CI does explicitly) ─────────────
::    Adjust LIBUV_ROOT to wherever you installed / built libuv for ARM64
set LIBUV_ROOT=C:\dependencies\libuv\install
copy %LIBUV_ROOT%\bin\uv.dll torch\lib\uv.dll

:: ── 5. Point CMake at APL (adjust APL_ROOT to your install path) ──────────────
set APL_ROOT=C:\arm-performance-libraries
set CMAKE_PREFIX_PATH=%APL_ROOT%;%CMAKE_PREFIX_PATH%

:: ── 6. Set build environment variables ───────────────────────────────────────

set CMAKE_BUILD_TYPE=Release
set BUILD_TYPE=Release

:: BLAS = APL  (matches BLAS_INFO=apl, LAPACK_INFO=apl)
set BLAS=APL
set ENABLE_APL=1
set USE_LAPACK=1
set USE_MKL=OFF

:: MKL-DNN OFF  (matches USE_MKLDNN=OFF)
set USE_MKLDNN=OFF

:: XNNPACK OFF  (absent from ARM64 CXX_FLAGS)
set USE_XNNPACK=OFF

:: Threading
set USE_OPENMP=ON

:: Distributed
set USE_GLOO=ON
set USE_MPI=OFF
set USE_NCCL=OFF

:: No GPU
set USE_CUDA=OFF
set USE_CUDNN=OFF
set USE_ROCM=OFF
set USE_XPU=OFF
set USE_XCCL=OFF

:: Profiling — Kineto without CUPTI / ROCTracer
set USE_KINETO=ON
set LIBKINETO_NOCUPTI=ON
set LIBKINETO_NOROCTRACER=ON
set LIBKINETO_NOXPUPTI=ON

:: Misc
set USE_NNPACK=OFF
set USE_GFLAGS=OFF
set USE_GLOG=OFF
set INSTALL_TEST=0
set BUILD_TEST=0
set DISTUTILS_USE_SDK=1

:: Version  (matches TORCH_VERSION=2.12.0)
set PYTORCH_BUILD_VERSION=2.12.0
set PYTORCH_BUILD_NUMBER=1

:: ── 7. Build wheel ────────────────────────────────────────────────────────────
python -m build --wheel --no-isolation --outdir dist\

:: ── 8. Install ────────────────────────────────────────────────────────────────
pip install dist\torch-2.12.0-*.whl
```

---

## Optional: sccache for Faster Rebuilds

The official CI wraps the compiler with [sccache](https://github.com/mozilla/sccache)
(`sccache-cl.exe` on x64) to cache object files across builds. This is a **build-time
speed optimization only** and does not affect the resulting binary or its config output.

```bat
:: Install sccache
winget install Mozilla.sccache

:: Add these lines before step 6 (Build wheel) in either script above:
set CMAKE_C_COMPILER_LAUNCHER=sccache
set CMAKE_CXX_COMPILER_LAUNCHER=sccache
sccache --start-server
sccache --zero-stats

:: After the build, inspect the cache hit rate:
sccache --show-stats
```

On a warm cache, sccache can cut rebuild times by 50–80 %.

---

## Verifying Your Build

After installing the wheel, confirm the config matches the official output:

```python
import torch
print(torch.__config__.show())
```

Key fields to check:

| Field | x64 expected | ARM64 expected |
|---|---|---|
| `BLAS_INFO` | `mkl` | `apl` |
| `LAPACK_INFO` | `mkl` | `apl` |
| `USE_MKLDNN` | `ON` | `OFF` |
| `USE_CUDA` | `0` / `OFF` | `OFF` |
| `USE_OPENMP` | `ON` | `ON` |
| `USE_GLOO` | `ON` | `ON` |
| CPU capability | `AVX2` | `DEFAULT` |
 
