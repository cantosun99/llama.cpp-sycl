# llama.cpp-sycl

llama.cpp with Intel GPU acceleration, packaged for Arch Linux.

Intel makes great GPUs. They also make it nearly impossible to use them on anything that isn't Ubuntu. The oneAPI toolchain comes with apt, zypper, and yum/dnf support, nothing for Arch. The AUR has the full `intel-oneapi-toolkit` package which is gigabytes of stuff you'll never use to actually run llama.cpp. If you want SYCL acceleration on Arch, you're basically on your own.

This package fixes that. It builds llama.cpp on your machine with full SYCL support at close to 10 GB less than the full kit.

Tested on a **Sparkle Intel Arc Pro B70 passive** running **Unsloth's Qwen3.6 27B Q6_K** on **CachyOS running the Linux 7.0.3-1-cachyos kernel**.

Note that this is my first package, please have patience. I greatly appreciate any help and hope to provide some value. You can contact me at privat@cantosun.de.

---

## Current news

Currently the Vulkan backend both supports MTP and with the [llama.cpp b9368 release](https://github.com/ggml-org/llama.cpp/releases/tag/b9368) also overtook SYCL in non-MTP speed on not all but most LLMs. As long as Intel doesn't release a 2026.0.1 or 2026.1.0 version of oneAPI, I have to, in full transparancy, recommend you to use the llama.cpp-vulkan package on the AUR to for example run Qwen 3.6 27B with MTP, which gets close to 30t/s in token generation on my B70.

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

The build takes a while depending on your CPU. This is normal. On my 245KF it takes about three minutes and is about 5.5 GB in total.

For further information, you can visit the llama.cpp documentation of the SYCL backend https://github.com/ggml-org/llama.cpp/blob/master/docs/backend/SYCL.md or the article by Intel https://www.intel.com/content/www/us/en/developer/articles/technical/run-llms-on-gpus-using-llama-cpp.html

---

## First-time setup

After installation, verify that everything is working before running your first model.

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
   bash: BASH_VERSION = 5.3.9(1)-release
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

Expected output (version may differ):

```
Intel(R) oneAPI DPC++/C++ Compiler 2026.0.0 (2026.0.0.20260331)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2026.0/bin/compiler
Configuration file: /opt/intel/oneapi/compiler/2026.0/bin/compiler/../icpx.cfg

```

### 4. Verify your GPU is detected by SYCL

```bash
sycl-ls
```

You should see at least one `level_zero:gpu` entry for your Intel GPU:

```
[level_zero:gpu][level_zero:0] Intel(R) oneAPI Unified Runtime over Level-Zero V2, Intel(R) Graphics [0xe223] 20.2.0 [1.15.37833]
```

### 5. Verify llama.cpp sees the GPU

```bash
/opt/llama.cpp-sycl/bin/llama-cli --list-devices
```

You should see your GPU listed as a SYCL device:

```
SYCL0: Intel(R) Graphics [0xe223] (32656 MiB, 31671 MiB free)
```

If all steps above produce output similar to the examples, you're ready to go.

---

## Daily usage

Every time you want to run llama.cpp, you need to load the oneAPI environment first, not just copy-paste your llama-server command as you would with other builds.

### Example 1: Loading the oneAPI environment and then running Qwen3.6 27B for precise coding on a B70

```bash
bash
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

### Examlple 2: Loading the oneAPI enviornment and then running Gemma 4 31B for writing on a B70

```bash
bash
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
