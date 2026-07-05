# llama.cpp-sycl

llama.cpp with Intel GPU acceleration, packaged for Arch Linux.

Intel makes great GPUs. They also make it nearly impossible to use them on anything that isn't Ubuntu. The oneAPI toolchain comes with apt, zypper, and yum/dnf support, nothing for Arch. The AUR has the full `intel-oneapi-toolkit` package which is gigabytes of stuff you'll never use to actually run llama.cpp. If you want SYCL acceleration on Arch, you're basically on your own.

This package fixes that. It builds llama.cpp on your machine with full SYCL support at close to *10 GB less than the full kit*.

---

## Current news

**Important: SYCL now supports MTP and beats llama.cpp-vulkan in PP by ~150 tok/s while only being ~1.5 tok/s slower in TG.**

Also SYCL has gotten a new release on the 1st of July with the Intel Intel® Deep Learning Essentials now being on version 2026.1.0 and the Intel® Deep Neural Network Library on 2026.0.1 so I expect the future llama.cpp releases to further optimize the speed of the SYCL backend.

Benchmarks ran with **llama.cpp b9873** on a **Sparkle Intel Arc Pro B70 passive** using Unsloth's **Qwen3.6 27B Q6_K** on **Arch Linux x86_64** with the **Linux 7.1.2-3-cachyos** kernel.

Since llama-bench doesn't support MTP, I used my recommended daily-usage flags, warmed the model up with "Say hi." and then benchmarked it for prompt processing with "Read the following text and then only reply with "ok" once you have done so." followed by the German Wikipedia entry for Black Sabbath converted to text. For token generation I also warmed it up with "Say hi." and benchmarked it with "Write a complete Snake game in a single Python file. Output only the code, no explanations, no commentary. Include: arrow key controls, collision detection, food spawning, score display, and a game over screen. Make it actually runnable." with thinking off.

| Backend | MTP | PP | TG |
| --- | --- | --- | --- |
| Vulkan | off | 317.32 tok/s | 16.92 tok/s |
| Vulkan | on | 313.88 tok/s | 33.40 tok/s |
| SYCL | off | 485.18 tok/s | 15.75 tok/s |
| SYCL | on | 423.76 tok/s | 31.64 tok/s |

---

## What this does

The PKGBUILD will:

1. Download the Intel Deep Learning Essentials and oneDNN installers directly from Intel (pinned to a specific version, updated manually per release)
2. Install the oneAPI toolchain temporarily during the build, then package it to `/opt/intel/oneapi/`
3. Clone [llama.cpp](https://github.com/ggml-org/llama.cpp) from source
4. Build it with SYCL enabled using Intel's `icx`/`icpx` compilers
5. Install shared libraries to `/opt/llama.cpp-sycl/lib/` with RPATH baked in, keeping them out of the global `/usr/lib` namespace
6. Install binaries to `/opt/llama.cpp-sycl/bin/`
7. Symlink all binaries into `/usr/bin/` so they are accessible system-wide without any environment setup

---

## Install

### Via AUR helper

```bash
yay -S llama.cpp-sycl
# or
paru -S llama.cpp-sycl
```

### Manual

```bash
git clone https://github.com/cantosun99/llama.cpp-sycl.git
cd llama.cpp-sycl
makepkg -si
```

The build takes a while depending on your CPU. This is normal. On my 270K it takes about two to three minutes and is about 6.0 GB installed in total.

For further information, you can visit the llama.cpp documentation of the SYCL backend https://github.com/ggml-org/llama.cpp/blob/master/docs/backend/SYCL.md or the article by Intel https://www.intel.com/content/www/us/en/developer/articles/technical/run-llms-on-gpus-using-llama-cpp.html

---

## First-time setup

After installation, verify that everything is working before running your first model. Versions may differ.

### 1. Use bash

The oneAPI `setvars.sh` script requires bash. If your default shell is fish or zsh, switch to bash:

```bash
bash
```

### 2. Load the oneAPI environment

```bash
source /opt/intel/oneapi/setvars.sh
```

Expected output:

```
:: initializing oneAPI environment ...
   bash: BASH_VERSION = 5.3.15(1)-release
   args: Using "$@" for setvars.sh arguments: 
:: ccl -- latest
:: compiler -- latest
:: debugger -- latest
:: dev-utilities -- latest
:: dnnl -- latest
:: dpl -- latest
:: mkl -- latest
:: mpi -- latest
:: pti -- latest
:: tbb -- latest
:: tcm -- latest
:: umf -- latest
:: oneAPI environment initialized ::
```

### 3. Verify the compiler

```bash
icpx --version
```

Expected output:

```
Intel(R) oneAPI DPC++/C++ Compiler 2026.1.0 (2026.1.0.20260617)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2026.1/bin/compiler
Configuration file: /opt/intel/oneapi/compiler/2026.1/bin/compiler/../icpx.cfg
```

### 4. Verify your GPU is detected by SYCL

```bash
sycl-ls
```

You should see at least one `level_zero:gpu` entry for your Intel GPU:

```
[level_zero:gpu][level_zero:0] Intel(R) oneAPI Unified Runtime over Level-Zero V2, Intel(R) Arc(TM) Pro B70 Graphics 20.2.0 [1.15.38646]
[level_zero:gpu][level_zero:1] Intel(R) oneAPI Unified Runtime over Level-Zero V2, Intel(R) Graphics 12.70.4 [1.15.38646]
[opencl:cpu][opencl:0] Intel(R) OpenCL, Intel(R) Core(TM) Ultra 7 270K Plus OpenCL 3.0 (Build 0) [2026.21.6.0.17_160000]
[opencl:gpu][opencl:1] rusticl, Mesa Intel(R) Graphics (BMG G31) OpenCL 3.0  [26.1.4-arch3.1]
[opencl:gpu][opencl:2] rusticl, Mesa Intel(R) Graphics (ARL) OpenCL 3.0  [26.1.4-arch3.1]
[opencl:gpu][opencl:3] Intel(R) OpenCL Graphics, Intel(R) Arc(TM) Pro B70 Graphics OpenCL 3.0 NEO  [26.22.38646]
[opencl:gpu][opencl:4] Intel(R) OpenCL Graphics, Intel(R) Graphics OpenCL 3.0 NEO  [26.22.38646]
```

### 5. Verify llama.cpp sees the GPU

```bash
/opt/llama.cpp-sycl/bin/llama-cli --list-devices
```

You should see your GPU listed as a SYCL device:

```
Available devices:
  SYCL0: Intel(R) Arc(TM) Pro B70 Graphics (32656 MiB, 31665 MiB free)
  SYCL1: Intel(R) Graphics (29115 MiB, 11642 MiB free)
```

If all steps above produce output similar to the examples, you're ready to go.

---

## Daily usage

Every time you want to run llama.cpp, you need to load the oneAPI environment first, not just copy-paste your llama-server command as you would with other builds.

### Example 1: Loading the oneAPI environment and then running Qwen3.6 27B for precise coding on a B70 *with* MTP

```bash
source /opt/intel/oneapi/setvars.sh
/opt/llama.cpp-sycl/bin/llama-server \
  -m /path/to/your/model/Qwen3.6-27B-Q6_K.gguf \
  --device SYCL0 \
  --n-gpu-layers 999 \
  --no-mmap \
  --flash-attn on \
  --jinja \
  --ctx-size 131072 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --port 8001
```

### Example 2: Loading the oneAPI environment and then running Qwen3.6 27B for precise coding on a B70 *without* MTP

```bash
source /opt/intel/oneapi/setvars.sh
/opt/llama.cpp-sycl/bin/llama-server \
  -m /path/to/your/model/Qwen3.6-27B-Q6_K.gguf \
  --device SYCL0 \
  --n-gpu-layers 999 \
  --no-mmap \
  --flash-attn on \
  --jinja \
  --ctx-size 131072 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --port 8001
```

### Examlple 3: Loading the oneAPI enviornment and then running Gemma 4 31B for writing on a B70 *with* MTP

```bash
source /opt/intel/oneapi/setvars.sh
/home/c/llama.cpp/build/bin/llama-server \
  -m /path/to/your/model/gemma-4-31B-it-Q6_K.gguf \
  --device SYCL0 \
  --n-gpu-layers 999 \
  --no-mmap \
  --flash-attn on \
  --jinja \
  --ctx-size 65536 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 1 \
  --top-p 0.95 \
  --top-k 64 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --port 8001
```

### Examlple 4: Loading the oneAPI enviornment and then running Gemma 4 31B for writing on a B70 *without* MTP

```bash
source /opt/intel/oneapi/setvars.sh
/home/c/llama.cpp/build/bin/llama-server \
  -m /path/to/your/model/gemma-4-31B-it-Q6_K.gguf \
  --device SYCL0 \
  --n-gpu-layers 999 \
  --no-mmap \
  --flash-attn on \
  --jinja \
  --ctx-size 65536 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 1 \
  --top-p 0.95 \
  --top-k 64 \
  --port 8001
```

---

## License

PKGBUILD and packaging: MIT. llama.cpp: MIT. Intel oneAPI components are subject to Intel's license terms.
