<p align="center">
  <img src="https://miro.medium.com/v2/resize:fit:1400/1*3wneegX3hSMGtC92UXSjKg.jpeg" alt="Llama.cpp Banner" width="100%">
</p>

This repository provides a complete guide and optimized scripts to run ultra-compact LLMs like **Gemma 3 270M** inside Android Termux, 
utilizing full GPU hardware acceleration via **Vulkan (Mesa Turnip Drivers)** and `llama.cpp`.

By bypassing standard Android restrictions, this setup achieves blazing-fast on-device inference without overheating the CPU.

## 🛠️ Requirements & Environment
* **Device:** Snapdragon SoC with Adreno GPU (Tested on Adreno 650)
* **OS:** Android (with MTE/Tagged Pointers active)
* **Environment:** Termux (Latest Version)

---

## 💾 Quick Installation

Instead of compiling for 45 minutes, we use the official pre-compiled Termux packages and inject the hardware-specific graphics layers.

### 1. Install Dependencies & Packages
Update your Termux repositories and install `llama-cpp` along with the necessary Vulkan wrappers:

```bash
pkg update && pkg upgrade -y
pkg install tur-repo -y
pkg install x11-repo -y
```
-
```
pkg install vulkan-tools vulkan-loader-generic mesa-vulkan-icd-freedreno -y
pkg update -y
```
-
```
pkg install llama-cpp llama-cpp-backend-vulkan -y
```

### 2. Verify GPU Detection
To ensure your phone's physical graphics card is used instead of software rendering (`llvmpipe`), run:

```bash
export VK_ICD_FILENAMES=\$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json
vulkaninfo --summary
```
*Expected Output:* `deviceName = Turnip Adreno (TM) XXX`.
If no drivers are installed, follow the link at the very bottom to complete all the necessary steps. 

---

## 💻 Model Setup (Gemma 3)

We use the optimized Unsloth Dynamic 8-bit quantization of Google's Gemma 3 270M model because it fits flawlessly into the mobile VRAM.

Download the GGUF file to your home directory:
```bash
mkdir -p models && cd models
wget https://huggingface.co/unsloth/gemma-3-270m-it-GGUF/resolve/main/gemma-3-270m-it-UD-Q8_K_XL.gguf
```

---

## Running the WebUI Server

To launch the integrated local WebUI interface powered entirely by your **Adreno GPU**, run the server by passing dynamic sampling flags tailored for small-parameter models:

```bash
VK_ICD_FILENAMES=\$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json \
KMP_AFFINITY=disabled \
EXECUTABLE_DISABLE_MTE=1 \
llama-server \
  -m ~/models/gemma-3-270m-it-UD-Q8_K_XL.gguf \
  -ngl 99 \
  --port 8080 \
  --host 127.0.0.1 \
  --temp 0.6 \
  --min-p 0.08 \
  --no-warmup \
  -c 2048
```

### Key Parameter Settings Used:
* `-ngl 99`: Offloads all model layers to the Vulkan GPU pipeline.
* `--min-p 0.08`: Aggressively prunes the massive 256k Gemma vocabulary to avoid hallucinations or gibberish.
* `EXECUTABLE_DISABLE_MTE=1`: Prevents Android's modern memory tagging system from crashing the linker.
* `--no-warmup`: Bypasses the OpenMP multi-threading topology bug (`Error #13`).

### Accessing the Chat Interface
Open your Android web browser and navigate to:
👉 **`http://127.0.0.1:8080`**

---

## 🐛 Troubleshooting Covered in this Repo
* **Pointer tag truncated / Clang Aborted:** Fixed by setting `EXECUTABLE_DISABLE_MTE=1`.
* **OMP: Error #13 Assertion Failure:** Bypassed via `--no-warmup` and `KMP_AFFINITY=disabled`.
* **Vulkan Info showing `llvmpipe`:** Solved by mapping the dedicated Turnip ICD backend file path.

---

# Tuning & Optimization for Qualcomm Adreno
## Adreno Vulkan Optimization Environment

Optimized environment variables for running graphics applications and AI inference (such as GGML/llama.cpp) on Qualcomm Adreno GPUs under Linux/Android (e.g., via Termux/PRoot). 

The configuration forces the use of Vulkan via the **Zink Gallium Driver** (OpenGL-to-Vulkan translation) and optimizes shader performance.

## Configuration Breakdown

These settings optimize the following areas:
* **GGML / Vulkan**: Prioritizes compute performance (`HIGH_OCCUPANCY`) and utilizes host memory to improve stability.
* **Zink (OpenGL over Vulkan)**: Enables the `zink` driver with optimized descriptor management (`lazy`).
* **Performance & Caching**: Disables V-Sync (`vblank_mode=0`), enables OpenGL threading, and configures a 2 GB shader cache for faster load times.
* **WSI / Display**: Forces `immediate` presentation mode for minimal latency
* (tearing may occur, optional use (`export MESA_VK_WSI_PRESENT_MODE=mailbox`) instead).
* **Tools**: Adds the `glslc` shader compiler (Shaderc) to the system path.

## How to Use

Simply copy these lines into your terminal before running your application, or add them permanently to your shell configuration:

```
# --- Adreno Vulkan ---
export VK_ICD_FILENAMES="$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json"
export GGML_VK_COMPUTE_OCCUPANCY_PRIORITY=high
export GGML_VK_PREFER_HOST_MEMORY=1
# export DISPLAY=:1
export GALLIUM_DRIVER=zink
export MESA_NO_ERROR=1
export vblank_mode=0
export MESA_GLSL_CACHE_DISABLE=0
export MESA_SHADER_CACHE_DISABLE=0
export MESA_SHADER_CACHE_MAX_SIZE=2G
export mesa_glthread=true
export ZINK_DESCRIPTORS=lazy
# export MESA_VK_WSI_PRESENT_MODE=mailbox
export MESA_VK_WSI_PRESENT_MODE=immediate
export ASAN_OPTIONS=allow_user_segv_handler=1
export MESA_SPIRV_LOG_LEVEL=warn
export PATH=$PATH:$HOME/shaderc/build/glslc/
# --- done
```

---

* You can verify that your optimized configurations are loaded into the operational landscape of your current terminal environment by using the query tools:
```
  echo $MESA_VK_WSI_PRESENT_MODE
```
*It must output: *immediate* to confirm the bypass of presentation synchronization locks.
#
* Checking the Shader Compiler Path Extension:
```
which glslc
```
*If you have compiled shaderc successfully in that directory,
it should return your precise path instead of a blank line.

---

## 🔎 Turnip Adreno Vulkan/Mesa Termux Driver
```
https://github.com/qapdex-maker/termux-notes/blob/main/adreno-vulkan/README.md
``` 
