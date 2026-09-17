# Linux / OmarchyOS LLM Inference Optimization — Forum & Community Research

Research date: 2026-09-16

## Sources Consulted
- r/LocalLLaMA, r/LocalLLM (Reddit)
- Level1Techs Forums (Machine Learning section)
- CachyOS Wiki
- Phoronix
- Omarchy community / blog posts
- GitHub repos (omarchy-ai, cachyos-whitedragon-ai-lab, gemma4-vulkan-cachyos)

---

## 1. GPU Offloading & Layer Splitting

### The `-ngl` Flag (llama.cpp)
```bash
# Offload everything that fits
./llama-server -m model-Q4_K_M.gguf -ngl 999

# Manual layer split for mixed GPU/CPU
./llama-server -m model-Q4_K_M.gguf -ngl 35
```

### MoE Model CPU Offloading (old GPU trick)
For MoE models on cards with limited VRAM (e.g., GTX 1080 8GB):
```bash
# Keep MoE experts of first N layers on CPU, rest on GPU
--n-cpu-moe 30 --n-gpu-layers 999
```
Source: https://mdda.net/blog/tech/dl/llama-cpp-moe-on-an-old-gtx-1080

### Tensor Split Across Uneven GPUs
```bash
--tensor-split 16,12   # GPU0 gets 16GB layers, GPU1 gets 12GB
```
Source: Level1Techs forum — "Breaking GPU compute down. Maximize VRAM"

---

## 2. Vulkan Backend for Old/Unsupported AMD GPUs

**Critical for old AMD cards** (RX 570 4GB, RX 580 8GB, Vega 64, Polaris):
- ROCm dropped support for gfx900 (Vega 64) — Vulkan is the only path
- CachyOS has RADV drivers built-in (Mesa)

Build on CachyOS:
```bash
# Install deps
sudo pacman -S vulkan-headers vulkan-icd-loader vulkan-tools vulkan-radeon shaderc cmake git

# Build llama.cpp with Vulkan
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build -DGGML_VULKAN=1 -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j$(nproc)
sudo cmake --install build --config Release
```
Source: https://github.com/RigelV/gemma4-vulkan-cachyos

### Community Result — RX 570 4GB on CachyOS:
```bash
./llama-server -m gemma-4-E2B-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 11435 \
  --ctx-size 8192 --n-gpu-layers 99 \
  --threads 4 --no-warmup --reasoning off -np 2
```
**56 tokens/sec at 8k context, 3.6GB VRAM usage**
Source: https://www.reddit.com/r/LocalLLM/comments/1td9bj9/

---

## 3. Quantization Strategies

### K-Quant Recommendation (GGUF)
| Type | Bits | Size (7B) | Quality | Use Case |
|------|------|-----------|---------|----------|
| Q4_K_M | 4.5 | ~4.1 GB | High | **Recommended default** |
| Q5_K_M | 5.5 | ~4.8 GB | Very High | Quality focused |
| Q6_K | 6.0 | ~5.5 GB | Excellent | Near-original |
| Q3_K_M | 3.3 | ~3.3 GB | Medium | Memory constrained |
| Q2_K | 2.5 | ~2.8 GB | Low | Extreme compression |

### Importance Matrix (iqm) for Better Low-Bit Quality
```bash
# Generate importance matrix
./llama-imatrix -m model-f16.gguf -f calibration.txt -o model.imatrix

# Apply with imatrix
./llama-quantize --imatrix model.imatrix model-f16.gguf model-q4_K_M.gguf Q4_K_M
```

### KV Cache Quantization (huge VRAM saver)
```bash
--cache-type-k turbo4 --cache-type-v turbo3
```
Source: https://mdda.net/blog/tech/dl/llama-cpp-moe-on-an-old-gtx-1080

---

## 4. Omarchy-Specific Setup

### Omarchy + AMD iGPU (Strix Halo / Ryzen AI Max+)
- 96GB allocated as VRAM/GTT out of 128GB unified memory
- llama.cpp server mode with OpenAI-compatible API
- Speculative decoding with DFlash2
- Source: https://goranpavlovic.ca/blog/local-agentic-coding

### Omarchy + NVIDIA Hybrid Laptop
- Install Ollama with VS Code Copilot integration
- Uses llama.cpp with CUDA backend
- Source: https://gist.github.com/n-studio/663ae3a4e4d41576436c72d4e77f12ee

### Omarchy M1 Mac — Neural Engine + GPU
- Vulkan backend requires `spirv-headers`: `sudo pacman -S spirv-headers`
- Source: https://joshuawarren.com/blog/m1-neural-engine-linux-gpu-llm

### omarchy-ai Meta-Repo
```bash
# One-command AI dev environment setup
curl -fsSL https://raw.githubusercontent.com/mitkox/omarchy-ai/main/boot.sh | bash
```
Features: llama.cpp + CUDA, Jupyter, model management, GPU monitoring
Source: https://github.com/mitkox/omarchy-ai

---

## 5. CachyOS-Specific Optimizations

### Kernel Advantages
- Linux Zen kernel (default) — optimized for desktop/compute workloads
- Advanced compilation: LTO, AutoFDO, kCFI
- Timer frequency: 1000Hz default (lower latency)
- PREEMPT_DYNAMIC: runtime-selectable preemption modes
- BFQ/mq-deadline I/O schedulers, ADIOS v3.2, BBR3

### Gaming/AI Env Vars (from CachyOS wiki)
```bash
# For Vulkan/compute workloads
export MESA_LOADER_DRIVER_OVERRIDE=zink   # fallback path
export RADV_PERFTEST=aco                  # AMD compiler opt
```

### Build llama.cpp with ROCm on CachyOS
```bash
sudo pacman -S rocm-hip-sdk rocblas hipblas
cmake -B build -DGGML_HIP=ON -DCMAKE_BUILD_TYPE=Release \
  -DAMDGPU_TARGETS="gfx1030;gfx1100;gfx1101" \
  -DCMAKE_INSTALL_PREFIX=/usr/local
cmake --build build --config Release -j$(nproc)
```
Source: https://github.com/meltingscales/cachyos-whitedragon-ai-lab

---

## 6. Linux Kernel & Boot Parameter Tweaks

### GRUB Kernel Parameters for LLM Inference
```
# /etc/default/grub — append to GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="... transparent_hugepage=madvise mitigations=off ttm.page_pool_size=27648000"
```
Then: `sudo grubby --update-kernel=ALL --args='ttm.page_pool_size=27648000'`

**ttm.page_pool_size**: Increases TTM page pool for GPU memory allocation (reported +10-15% on AMD)
**mitigations=off**: Bypasses Spectre/Meltdown patches — 10-15% throughput uplift (security tradeoff)
**nohpet**: Disables HPET — lower interrupt latency

Source: https://forum.level1techs.com — GLM-4.7 on x3950 X6 guide

### Sysctl Tuning
```bash
# io_uring for small models (<10B params)
fs.aio-max-nr = 1048576
kernel.io_uring_disabled = 0

# Memory
vm.nr_hugepages = 512           # 1GB hugepages for model weights
vm.transparent_hugepages.enabled = madvise
vm.transparent_hugepages.defrag = madvise
vm.memory_fragmentation_index = 1

# Scheduler (disable NUMA balancing for 70B+ models)
kernel.sched_min_granularity_ns = 10000000
kernel.sched_wakeup_granularity_ns = 15000000
kernel.sched_numa_balancing = 0

# Network
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
```
Source: https://johal.in/surprising-truth-linux-kernel-68-vs-llm-inference

### THP Drop Caches (before running inference)
```bash
sudo sync
echo 3 | sudo tee /proc/sys/vm/drop_caches
echo always | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

---

## 7. Speculative Decoding (DFlash2 / MTP)

### DFlash2 — Modern Speculative Decoding
- Works with Qwen 3.8 27B, GLM 5.3 on gfx906
- 250k context on 40GB VRAM (2x Radeon VII + GTX 3080)
- Source: https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906/254257

### MTP (Multi-Token Prediction) Caveat
- Only helps if draft model tensors are on GPU
- Gemma 4 26B-A4B: **must** use `--override-tensor-draft "token_embd\.weight=CUDA0"`
- Without this, MTP adds PCIe traffic and barely helps
```bash
--n-cpu-moe 21 --override-tensor-draft "token_embd\.weight=CUDA0"
```
Source: https://mdda.net/blog/tech/dl/llama-cpp-moe-on-an-old-gtx-1080

---

## 8. CPU-Only Extreme Optimization

### x3950 X6 (2015, 144 cores, 1TB RAM) — GLM-4.7 Q8_0
- BIOS: Disable hyper-threading, tweak power management
- NTPs: `numa distribute`, 64 threads
- Flags: `-gr` (graph reuse), `-rtr` (RoPE tweaks)
- Batch: `--batch-size 2048 --ubatch-size 2048 -amb 2048`
- Result: ~7.5 tok/s generation, ~23 t/s combined RAG
Source: https://postl.ai/2025/12/29/glm47on3950x6/

### Key Env Vars for CPU
```bash
export OMP_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
export MKL_NUM_THREADS=1
export BLIS_NUM_THREADS=1
export VECLIB_MAXIMUM_THREADS=1
export OMP_WAIT_POLICY=active
```

---

## 9. Kernel 6.8+ LLM-Specific Features

| Feature | Impact |
|---------|--------|
| io_uring batch recv/send | +22% throughput for <10B models |
| Split-LRU vmstat | -34% memory fragmentation, -85% OOM kills |
| THP on-demand defrag | +18% faster weight loading for 70B+ |
| userfaultfd write-protect | +29% faster model hot-reload |
| EEVDF scheduler regression | -7% p99 latency for 70B+ on NUMA (disable NUMA balancing) |

Source: https://johal.in/surprising-truth-linux-kernel-68-vs-llm-inference

---

## 10. Key Community Scripts & Repos

| Repo | Purpose |
|------|---------|
| `mitkox/omarchy-ai` | One-command AI dev environment on Omarchy |
| `RigelV/gemma4-vulkan-cachyos` | Vulkan build for old AMD on CachyOS |
| `meltingscales/cachyos-whitedragon-ai-lab` | llama.cpp ROCm install for CachyOS |
| `milpster/gfx906-llama-cpp` | Fork for Radeon VII/MI50/MI60 + mixed ROCm+Vulkan |
| `iacopPBK/llama.cpp-gfx906` | Primary gfx906 fork with DFlash2 support |
| `jakubivacek/InstallationCachyOS` | Full CachyOS gaming/AI setup guide |

---

## 11. Critical Pitfalls (Community-Reported)

1. **CachyOS AUR llama-vulkan** — VRAM limits immediately; use official Ubuntu binaries instead
2. **AMD Polaris (RX 570/580) — GFX 8** — Vulkan backend only; ROCm doesn't support
3. **glIBC 2.41 + CUDA 12.9** — patch `math_functions.h` or build fails
4. **llama.cpp default CPU threads = core_count/2** — often hurts performance; use `nproc` or `nproc-2`
5. **gemma-4 26B-A4B tied LM head** — MTP embedding table must be forced to GPU
6. **vLLM 0.23.x + ROCm 7.14** — 50% regression on FP8 MoE models (Qwen3.6)
7. **OpenVINO on Intel Arc** — only works with flash-attn=1; crashes without it
8. **Omarchy doesn't install spirv-headers** — Vulkan build of llama.cpp fails without them

---

## 12. Quick-Start: Old GPU on CachyOS/Omarchy

```bash
# 1. Install deps
sudo pacman -S git cmake ninja vulkan-radeon vulkan-headers vulkan-icd-loader shaderc

# 2. Build llama.cpp
git clone https://github.com/ggerganov/llama.cpp && cd llama.cpp
cmake -B build -DGGML_VULKAN=1 -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j$(nproc)
sudo cmake --install build --config Release

# 3. Download model (e.g., Gemma 4 2B Q4_K_M)
curl -L -o gemma-4-E2B-it-Q4_K_M.gguf \
  "https://huggingface.co/..."

# 4. Run
llama-server -m gemma-4-E2B-it-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 \
  -ngl 99 --ctx-size 8192 \
  --threads $(nproc) --no-warmup
```
