# llama.cpp-sycl

llama.cpp with Intel GPU acceleration, packaged for Arch Linux.

Intel makes great GPUs. They also make it nearly impossible to use them on anything that isn't Ubuntu. The oneAPI toolchain comes with apt, zypper, and yum/dnf support, nothing for Arch. The AUR has the full `intel-oneapi-toolkit` package which is gigabytes of stuff you'll never use to actually run llama.cpp. If you want SYCL acceleration on Arch, you're basically on your own.

This package fixes that. It ships the essential oneAPI components and builds llama.cpp on your machine with full SYCL support at close to 10 GB less than the full kit.

Tested on a **Sparkle Intel Arc Pro B70 passive** running **Unsloth's Qwen3.6 27B Q6_K** on **CachyOS running the Linux 7.0.3-1-cachyos kernel**.

Note that this is my first package, please have patience. I greatly appreciate any help and hope to provide some value. You can contact me at privat@cantosun.de.

---

## What this does

The PKGBUILD will:

1. Download and extract a bundled oneAPI runtime to `/opt/intel/oneapi/`
2. Clone [llama.cpp](https://github.com/ggml-org/llama.cpp) from source
3. Build it with SYCL enabled using Intel's `icx`/`icpx` compilers
4. Install all binaries to `/usr/bin/`

The build runs on your machine so you always get the latest llama.cpp. The oneAPI bundle is the hard part to get on Arch, and that's what this package provides.

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

The build takes a while depending on your CPU. This is normal.

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
:: ccl -- latest
:: compiler -- latest
:: dnnl -- latest
:: mkl -- latest
:: tbb -- latest
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
```

### 4. Verify your GPU is detected by SYCL

```bash
sycl-ls
```

You should see at least one `level_zero:gpu` entry for your Intel GPU:

```
[level_zero:gpu][level_zero:0] Intel(R) oneAPI Unified Runtime over Level-Zero V2, Intel(R) Graphics [0xe223]
```

### 5. Verify llama.cpp sees the GPU

```bash
llama-cli --list-devices
```

You should see your GPU listed as a SYCL device:

```
SYCL0: Intel(R) Graphics [0xe223] (32656 MiB, 31808 MiB free)
```

If all steps above produce output similar to the examples, you're ready to go.

---

## Daily usage

Every time you want to run llama.cpp, you need to load the oneAPI environment first.

### Loading the oneAPI environment

```bash
bash
source /opt/intel/oneapi/setvars.sh
```

### Example: running Qwen3.6 27B

```bash
./build/bin/llama-server \
  -m ~/Qwen3.6-27B-Q6_K.gguf \
  --device SYCL0 \
  -ngl 999 \
  --ctx-size 100000 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --min-p 0.00 \
  --presence-penalty 0.0 \
  --port 8001
```

---

## License

PKGBUILD and packaging: MIT. llama.cpp: MIT. Intel oneAPI components are subject to Intel's license terms.

If an Intel lawyer reads this, I'm new to this and just want to help improve the software situation of the Arc GPUs that I love so much. No harm intended.
