# llama.cpp-sycl

llama.cpp with Intel GPU acceleration, packaged for Arch Linux.

Intel makes great GPUs. They also make it nearly impossible to use them on anything that isn't Ubuntu. The oneAPI toolchain comes with apt, zypper, and yum/dnf support, nothing for Arch. The AUR has the full `intel-oneapi-toolkit` package which is gigabytes of stuff you'll never use to actually run llama.cpp. If you want SYCL acceleration on Arch, you're basically on your own.

This package fixes that. It builds llama.cpp on your machine with full SYCL support at close to *10 GB less than the full kit*.

---

## Current news

(August 15th) UD-Q6_K_XL fits a context of 131072 with barely 300MB VRAM breathing room into a B70, Q6_K leaves 3GB. UD-5_K_XL can be used if you need the full 262144 context, anything below 6-bit is usually not acceptable quality in my opinion, you have to work with what you have tho. Considering UD-Q5_K_XL fits full Q8_0 context, I consider this to be the smallest sensible quant and because Q8_0 already eats up 29GB with barely any room left for context, I don't see any point in a quant higher than UD-Q6_K_XL.

(August 14th, midnight) **I finally got done with a lot of testing and I have to say, Qwen3.8 27B is genuinely insane. Not one failed tool call, insanely time-consuming reasoning and very token-hungry, but in the end it's worth it because it perfectly one-shot most of the tasks. Updated my recommendations, off to bed now, have fun trying it out!**

(August 14th) **SYCL got a lot of love recently and currently gets peaks of 36 tok/s TG and 900 tok/s PP with Qwen3.8 27B which beats llama.cpp-vulkan by a long shot.**

(August 11th) The AUR is finally back again!

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

In my opinion Qwen3.8-27B is the best model you can currently run and it's not even remotely close. Gemma 4 and Qwen3.6 are outdated, Muse Glimmer has the dumbest relase date I've heard in my life, you can't just launch an inferior model to Qwen3.6 in the same week that Qwen3.8 releases. There might be use-cases for Nemotron 3.5 Lightning, if you depend on fast and accurate tool calls for example.

[Unsloth's guide how to run Qwen3.8-27B](https://unsloth.ai/docs/models/qwen3.8)
[Unslot's Hugging Face Repo for Qwen3.8-27B](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)

### Example: Loading the oneAPI environment and then running Qwen3.8-27B on a single B70 with MTP

Warning: tight fit with 300MB VRAM breathing room.

```bash
source /opt/intel/oneapi/setvars.sh
/opt/llama.cpp-sycl/bin/llama-server \
  --model /path/to/your/model/Qwen3.8-27B-UD-Q6_K_XL.gguf \
  --device SYCL0 \
  --n-gpu-layers 999 \
  --load-mode none \
  --flash-attn on \
  --jinja \
  --reasoning-preserve \
  --ctx-size 131072 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 1.0 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --presence-penalty 0.0 \
  --repeat-penalty 1.0 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --port 9931
```

### Setting a function in fish so you just have to type "qwen" in the terminal to launch you llama-server

Create a file called "/home/yourusername/.config/fish/functions/qwen.fish" and paste the following in there, obviously with the correct path.

```
function qwen
    bash -c "source /opt/intel/oneapi/setvars.sh && /opt/llama.cpp-sycl/bin/llama-server --model /path/to/your/model/Qwen3.8-27B-UD-Q6_K_XL.gguf --device SYCL0 --n-gpu-layers 999 --load-mode none --flash-attn on --jinja --reasoning-preserve --ctx-size 131072 --cache-type-k q8_0 --cache-type-v q8_0 --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.00 --presence-penalty 0.0 --repeat-penalty 1.0 --spec-type draft-mtp --spec-draft-n-max 2 --port 9931"
end
```

---

## License

PKGBUILD and packaging: MIT. llama.cpp: MIT. Intel oneAPI components are subject to Intel's license terms.
