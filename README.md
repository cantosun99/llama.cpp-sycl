# llama.cpp-sycl

llama.cpp with Intel GPU acceleration, packaged for Arch Linux.

Intel makes great GPUs. They also make it nearly impossible to use them on anything that isn't Ubuntu. The oneAPI toolchain comes with apt, zypper, and yum/dnf support, nothing for Arch. The AUR has the full `intel-oneapi-toolkit` package which is gigabytes of stuff you'll never use to actually run llama.cpp. If you want SYCL acceleration on Arch, you're basically on your own.

This package fixes that. It builds llama.cpp on your machine with full SYCL support at close to *10 GB less than the full kit*.

---

## Current stats

**SYCL got a lot of love recently and with my recommended settings it currently gets peaks of 37 tok/s TG and 2200 tok/s PP (with adjusted batch and ubatch) with Qwen3.8 27B which beats llama.cpp-vulkan by a long shot.** Tested with a Sparkle Intel Arc Pro B70 at 275W, using the Linux 7.2.0-1-cachyos kernel.

## Current news

(August 20th) Unsloth released new Unsloth Dynamic v3.0 quants for Qwen3.8-27B which currently are my recommendation. If your gguf is older than August 19th, you should re-download it. I have updated the recommendations with two examples.

Example 1: UD-Q6_K_XL, the highest quant you can run on a B70 for maximum quality with a small but usally sufficient context of 100000 tokens. Highest quality but still enough context to handle the xhigh reasoning for short tasks that require accuracy and attention to detail. This should be your "daily-driver" for 90% of tasks.

Example 2: UD-Q4_K_XL, the highest quant you can run on a B70 that still allows you to use the full 262144 token context. Slightly reduced quality but a huge amount of context, ideal for tasks that don't require as much accuracy or intensive reasoning and instead work with a ton of data.

(August 14th, midnight) **I finally got done with a lot of testing and I have to say, Qwen3.8 27B is genuinely insane. Not one failed tool call, insanely time-consuming reasoning and very token-hungry, but in the end it's worth it because it perfectly one-shot most of the tasks. Updated my recommendations, off to bed now, have fun trying it out!**

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

## Recommended usage

Every time you want to run llama.cpp, you need to load the oneAPI environment first, not just copy-paste your llama-server command as you would with other builds.

In my opinion Qwen3.8-27B is the best model you can currently run and it's not even remotely close. Gemma 4 and Qwen3.6 are outdated, Muse Glimmer has the dumbest relase date I've heard in my life, you can't just launch an inferior model to Qwen3.6 in the same week that Qwen3.8 releases. There might be use-cases for Nemotron 3.5 Lightning, if you depend on fast and accurate tool calls for example.

**Both of these are just examples from my experience, please do your own research and experimentation as it's both fun and rewarding to learn about this technology and adapt it to your own needs.**

For example with my Intel 270K I can set the threads flag to 24 but not everyone has that many cores. Maybe you prefer to use medium reasoning instead of the regular xhigh. Maybe you rely more, maybe you rely less on PP speed so you wanna adjust the batch and ubatch settings up to for example --batch-size 4096 and --ubatch-size 2048 and reduce the --ctx-size by around 30000-40000 for twice the PP speed but less available context. Maybe your tasks are likely what the model is trained on so you can set the draft tokens to 4, maybe you do more novel work and should keep it at 2. You are running Arch after all so please experiment!

[Unsloth's guide how to run Qwen3.8-27B](https://unsloth.ai/docs/models/qwen3.8)
[Unslot's Hugging Face Repo for Qwen3.8-27B](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)

### Example 1: UD-Q6_K_XL, the highest quant you can run on a B70 for maximum quality with a small but usally sufficient context of 100000 tokens. Highest quality but still enough context to handle the xhigh reasoning for short tasks that require accuracy and attention to detail. This should be your "daily-driver" for 90% of tasks.

```bash
source /opt/intel/oneapi/setvars.sh
/opt/llama.cpp-sycl/bin/llama-server \
  --model /path/to/your/model/Qwen3.8-27B-UD-Q6_K_XL.gguf \
  --device SYCL0 \
  --threads 24 \
  --load-mode none \
  --flash-attn on \
  --jinja \
  --reasoning-preserve \
  --ctx-size 131000 \
  --cache-type-k q8_0 \
  --cache-type-v q5_1 \
  --temp 1.0 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --presence-penalty 0.0 \
  --repeat-penalty 1.0 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --spec-draft-type-k q8_0 \
  --spec-draft-type-v q8_0 \
  --api-key llama.cpp-sycl \
  --port 9931
```

At the point of loading, this will use 29.481 GB of VRAM.

#### Setting a function in fish so you just have to type "qwen" in the terminal to launch you llama-server

Create a file called "/home/yourusername/.config/fish/functions/qwen.fish" and paste the following in there, obviously with the correct path.

```
function qwen
    bash -c "source /opt/intel/oneapi/setvars.sh && /opt/llama.cpp-sycl/bin/llama-server --model /path/to/your/model/Qwen3.8-27B-UD-Q6_K_XL.gguf --device SYCL0 --n-gpu-layers 999 --threads 24 --load-mode none --flash-attn on --jinja --reasoning-preserve --ctx-size 131000 --cache-type-k q8_0 --cache-type-v q5_1 --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.00 --presence-penalty 0.0 --repeat-penalty 1.0 --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-type-k q8_0 --spec-draft-type-v q8_0 --api-key llama.cpp-sycl --port 9931"
end
```

### Example 2: UD-Q4_K_XL, the highest quant you can run on a B70 that still allows you to use the full 262144 token context. Slightly reduced quality but a huge amount of context, ideal for tasks that don't require as much accuracy or intensive reasoning and instead work with a ton of data.

```bash
source /opt/intel/oneapi/setvars.sh
/opt/llama.cpp-sycl/bin/llama-server \
  --model /path/to/your/model/Qwen3.8-27B-UD-Q4_K_XL.gguf \
  --device SYCL0 \
  --threads 24 \
  --load-mode none \
  --flash-attn on \
  --jinja \
  --reasoning-preserve \
  --ctx-size 262144 \
  --cache-type-k q8_0 \
  --cache-type-v q5_1 \
  --temp 1.0 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --presence-penalty 0.0 \
  --repeat-penalty 1.0 \
  --spec-type draft-mtp \
  --spec-draft-n-max 2 \
  --spec-draft-type-k q8_0 \
  --spec-draft-type-v q8_0 \
  --api-key llama.cpp-sycl \
  --port 9931
```

At the point of loading, this will use 26.652 GB of VRAM.

#### Setting a function in fish so you just have to type "qwenmaxctx" in the terminal to launch you llama-server

Create a file called "/home/yourusername/.config/fish/functions/qwenmaxctx.fish" and paste the following in there, obviously with the correct path.

```
function qwenmaxctx
    bash -c "source /opt/intel/oneapi/setvars.sh && /opt/llama.cpp-sycl/bin/llama-server --model /path/to/your/model/Qwen3.8-27B-UD-Q4_K_XL.gguf --device SYCL0 --n-gpu-layers 999 --threads 24 --load-mode none --flash-attn on --jinja --reasoning-preserve --ctx-size 262144 --cache-type-k q8_0 --cache-type-v q5_1 --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.00 --presence-penalty 0.0 --repeat-penalty 1.0 --spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-type-k q8_0 --spec-draft-type-v q8_0 --api-key llama.cpp-sycl --port 9931"
end
```

---

## License

PKGBUILD and packaging: MIT. llama.cpp: MIT. Intel oneAPI components are subject to Intel's license terms.
