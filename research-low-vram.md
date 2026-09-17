# OmarchyOS & Low-VRAM AI Agent Techniques for GTX 1650 (4GB)

Sources: Linux/omarchy forums, community benchmarks, arxiv papers, GitHub repos (2025-2026)

---

## 1. OmarchyOS-Specific Setup

**GTT Memory Allocation (Limine bootloader)**
- Omarchy on Linux Zen kernel: set dedicated VRAM to minimum (512MB), let GTT (Graphics Translation Tables) allocate dynamically from system RAM
- iGPU can address full system RAM pool — critical for APU-based old hardware
- Reference: Goran Pavlović's Strix Halo setup (AMD APU, 100GB unified memory)

**Speculative Decoding + llama.cpp**
- Omarchy community runs llama.cpp in server mode with OpenAI-compatible API
- MTP (Multi-Token Prediction) speculative decoding: draft model predicts N tokens, main model verifies — 2-5x speedup on old GPUs

---

## 2. Quantization (4GB VRAM Budget)

| Quant | Size Ratio | Quality | Use Case |
|-------|-----------|---------|----------|
| Q8_0  | ~1.0x     | Near-lossless | Sub-2B models |
| Q6_K  | ~0.75x    | Near-lossless | Small models |
| Q5_K_M| ~0.69x    | Very good | Sweet spot for 1-3B |
| Q4_K_M| ~0.75     | Good, minor loss | Standard 4GB |
| Q3_K_M| ~0.81     | Noticeable degradation | Last resort for 7B |
| Q2_K  | ~0.87     | Significant loss | Avoid |

**TurboQuant KV cache**: 4× KV cache compression — Llama 3.1 8B Q4 at 64K ctx drops from 12.4GB to 8.6GB

---

## 3. GPU Offloading (llama.cpp flags)

```bash
# Auto-fit: llama.cpp calculates max layers that fit
./llama-server -m model.gguf -ngl 99 --fit on

# Manual tuning for GTX 1650 4GB:
# - Start at -ngl 15, increase until OOM
# - GTX 1650 typically handles 15-25 layers of a 7B Q4_K_M
# - Monitor: nvidia-smi, watch for 3.8GB/4GB usage

# MoE expert offloading (biggest win for MoE models):
--n-cpu-moe 32   # Keep first N layers' experts on CPU
```

**Key insight**: MoE models (Qwen3.5-35B-A3B, DeepSeek-V2, Kimi K2) only activate 2-8 of 256 experts per token — expert weights can live in RAM and stream over PCIe.

---

## 4. Swap-Based / Tiered Inference

### DeepswapLLM (GitHub: apolloraines)
- Three-tier: GPU VRAM → Pinned CPU RAM → NVMe Disk
- Zero-allocation double buffering: pre-allocated GPU buffers reused
- CUDA stream prefetch: loads layer N+1 while computing layer N
- MoE expert-level offloading: only loaded experts move to GPU
- Sparse block compression: up to 7× reduction for sparse layers
- Honest verdict: "If quantized model fits in VRAM, run that instead"

### VITRIOL (GitHub: Randozart)
- VRAM extension layer for llama.cpp
- Page-locked host RAM, GPU reads experts via PCIe DMA directly
- Chimera mode: CUDA for MoE experts + Vulkan for dense ops
- GTX 1070 Ti (8GB) ran Qwen3.6-35B-A3B at 23.3 tok/s with ~1.6GB VRAM
- **No AVX2 CPU required** — CPU is orchestrator only

### ORE Kernel (GitHub: Nithin-dot-k)
- Agent context paging to NVMe SSD
- Semaphore-based GPU leasing for multi-agent swarms
- KV cache frozen to disk, restored on next request
- 1,440% speedup on agent context switches vs recalculating

### Pie Framework (arxiv 2504.08791)
- Performance-transparent swapping: overlaps compute with PCIe transfer
- Adaptive expansion: dynamically adjusts CPU memory allocation
- Built on vLLM; zero latency impact when configured correctly

### VRAMSwapper (GitHub: ayinedjimi)
- LRU/LFU/Priority/Hybrid eviction strategies
- Unified memory manager with pool allocator
- Automatic prefetch queue

---

## 5. Distributed Inference (prima.cpp)

**Pipelined-Ring Parallelism (PRP)**
- Devices connected in ring; each handles a window of layers per round
- Prefetching overlaps disk I/O with compute
- Halda scheduler: optimal layer partitioning across heterogeneous devices
- Default cluster: 4 consumer devices with 37GB aggregate RAM/VRAM → runs 70B model
- Results: 674ms/token for 70B, 26 tok/s for 32B with speculative decoding
- Cross-platform: Linux, macOS, Android (Termux), HarmonyOS

**GTX 1650 application**: Could join a home cluster as a worker node, contributing GPU layers while other devices handle CPU layers.

---

## 6. GTX 1650 Specific Recommendations

**Hardware profile**: 4GB GDDR5, 192-bit bus, ~192 GB/s bandwidth, PCIe 3.0 x16, Pascal architecture (CC 6.1)

**Fitting models on 4GB**:
- LFM2.5-2.6B Q8_0: ~3.5GB, ~930 tok/s ✓
- Qwen3 0.6B Q8: ~2.6GB, ~285 tok/s ✓
- Qwen2.5 3B Q4_K_M: ~2.0GB, ~160 tok/s ✓
- Llama 3.2 3B Q4_K_M: ~2.0GB, ~160 tok/s ✓
- Phi-3.5-mini 3.8B Q3_K_S: ~3.9GB, ~255 tok/s ✓
- Qwen2.5-7B Q2_K: ~3.8GB, ~142 tok/s (quality degraded)
- Qwen3.5-35B-A3B Q4_K_M with --n-cpu-moe: ~19GB disk, ~5.8GB VRAM, ~20 tok/s ✓

**Critical settings**:
```bash
--no-mmap --mlock      # Prevent OS from evicting model pages
--cache-type-k q4_0    # Quantize KV cache
--cache-type-v q4_0
--fit on               # Auto-fit layers to VRAM
--mlock                # Lock memory to prevent swapping
```

**Bottleneck**: Memory bandwidth, not VRAM size. GTX 1650's 192 GB/s is ~3× slower than RTX 3060's 360 GB/s — expect ~40-60% of same-card throughput.

---

## 7. Cutting-Edge Techniques

1. **Hybrid CPU-GPU scheduling** (KTransformers): Intel AMX/AVX-512 for CPU expert compute, GPU for attention — runs 671B MoE on single 24GB GPU + 512GB RAM
2. **SageAttention2++**: INT8 QK + FP8 PV attention kernels — 2-5× speedup over FlashAttention, needs CUDA ≥ 12.8 (GTX 1650 = CC 6.1, won't benefit)
3. **Flash-Next** (Qwen 3.6 177B MoE): 20 tok/s on GTX 1650 with 38 expert layers in DDR4-2133, zero SSD reads
4. **RotorQuant**: Gemma-specific KV cache quantization — enables 128K context on 8GB cards
5. **ik_llama.cpp fork**: Specialized CPU kernels for MoE, better than vanilla llama.cpp on CPU-side expert compute
6. **Speculative drafting**: Small draft model on GPU, large model verifies — 2-5× throughput boost

---

## 8. What Doesn't Work on 4GB

- FLUX/DreamBooth image generation (needs 6-12GB minimum)
- Whisper large-v3 transcription (needs 3GB+ just for model)
- 7B models at Q4 without CPU offload (needs ~5GB)
- Long context (>8K) without KV cache quantization
- Multi-model concurrent serving

---

## Key Papers & Sources

- [prima.cpp arxiv 2504.08791](https://arxiv.org/abs/2504.08791) — Distributed on-device inference
- [Pie: Pooling CPU Memory for LLM Inference](https://arxiv.org/html/2411.09317) — Performance-transparent swapping
- [Maximum Capability from Minimum Silicon](https://local-ai-zone.github.io/blog/low-resource-ai-agents-research-paper.html) — 8GB VRAM agent guide
- [VITRIOL GitHub](https://github.com/Randozart/VITRIOL) — PCIe DMA VRAM extension
- [DeepswapLLM GitHub](https://github.com/apolloraines/DeepswapLLM) — Three-tier layer swapping
- [Goran Pavlović: Local Agentic Coding on Omarchy](https://goranpavlovic.ca/blog/local-agentic-coding) — Omarchy + Zen kernel setup
- [inferencerig.com: Best GGUF for 4GB](https://inferencerig.com/setup/best-gguf-models-for-4gb-vram-tested-on-low-end-gpus) — Tested models on GTX 1650
|- [fitmyllm.com: Running AI on Old GPUs](https://fitmyllm.com/blog/running-ai-on-old-gpus) — GTX 1060/1650 optimization guide

---

## 9. Kernel-Level Optimizations (2025-2026)

### Lazy Unmap Flush (LUF)
- **What**: Defers TLB flushes until folios are unmapped and freed
- **Impact**: 97% fewer TLB shootdowns, ~4.5% faster llama.cpp
- **Status**: RFC patches on LKML, not yet mainline
- **Source**: Phoronix, Feb 2025

### TLB Flush Batching During Reclaim
- **What**: Batches TLB flushes for dirty folios — one IPI per PMD instead of per page
- **Impact**: 26.9% throughput improvement on multi-core systems
- **Author**: Tencent engineer Zhang Peng, March 2026

### MGLRU Tuning
- CachyOS ships enhanced MGLRU patches
- Linux 7.3 keeps executable folios mapped longer
- Reduces page faults during model loading

### AutoFDO + Propeller
- Kernel-level profile-guided optimization
- ~10% throughput improvement (CachyOS default)
- AutoFDO: uses runtime profiling data for optimization
- Propeller: Intel CPU optimization (not applicable to NVIDIA GPU)

---

## 10. GGUF Quantization Advances (2025-2026)

### Beyond IQ2_XXS — What Actually Works on 4GB
- **IQ2_S** (2.5 bpw, ~9GB for 27B): Better quality-per-bit than IQ2_XXS; requires importance matrix (`llama-quantize` with imatrix). Use for models >13B where IQ2_XXS is too lossy.
- **IQ3_XXS** (3.06 bpw): The sweet spot for aggressive quantization — matches BF16 on AIME25 at 10.1GB for Qwen3.8-27B (GSQ-RCO). DASLab's gradient-based tensor-level allocation beats fixed recipes.
- **Unsloth Dynamic v2.0**: Per-tensor bit-width selection (Q4_NL/Q5.1/Q5.0/Q4.1/Q4_0) guided by gradient magnitude; 20-40% faster than fixed-bit quants with minimal quality loss.

### Importance Matrix (imatrix)
- Generates per-tensor scaling based on activation magnitudes
- Improves IQ2_S/IQ3_XXS quality by 2-5% on benchmarks
- Command: `llama-quantize --imatrix imatrix.dat model.gguf model-imatrix.gguf`

---

## 11. MoE Models for 4GB VRAM

### Models That Fit at Q4 Quantization
**Dense models** (simple, predictable):
| Model | Q4_K_M VRAM | Notes |
|---|---|---|
| Qwen3 0.6B | ~0.5 GB | Barely uses any VRAM |
| Qwen3 1.7B | ~1.1 GB | Comfortable headroom |
| Qwen3 4B | ~2.5 GB | Best quality-per-VRAM sweet spot |
| Qwen3 8B | ~4.6 GB | Tight fit, minimal KV cache room |

**Small MoE models** (sparse activation):
| Model | Params (total/active) | Q4_K_M VRAM | Notes |
|---|---|---|---|
| IBM Granite 3.1 MoE 1B | 1.33B | ~822 MB | Designed for low-latency edge |
| Qwen3 0.6B MoE | 0.6B/0.1B active | ~0.5 GB | Minimal expert activation |

### FATE-llama.cpp: Expert Offloading with Cross-Layer Prefetch
- **What**: GPU-optimized MoE expert offloading for consumer GPUs
- **Key technique**: Offload experts to CPU RAM, prefetch next expert during current computation
- **Cross-layer prefetch**: Overlaps PCIe transfer with computation
- **Impact**: 3-5× speedup vs vanilla llama.cpp for MoE models on 4GB VRAM
- **Source**: GitHub ongum/llama-moe-cache, April 2026

---

## 12. Vulkan Backend for Old NVIDIA GPUs

### Why Vulkan on Old NVIDIA?
- CUDA 12.x dropped support for CC < 5.0 (GTX 1650 = CC 6.1, still supported)
- But Vulkan backend is often faster than CUDA on old GPUs
- **Result**: 37% faster on GTX 1060, similar gains expected on GTX 1650

### Installation (Omarchy/Arch)
```bash
# Install from AUR (updated more frequently than extra repo)
yay -S llama.cpp-vulkan

# Or build from source with:
cmake -B build -DGGML_VULKAN=ON
```

### Vulkan vs CUDA on GTX 1650
- Vulkan: better driver maturity on old NVIDIA cards
- CUDA: more optimization for modern architectures
- **Recommendation**: Test both, Vulkan often wins on Turing (SM75)

---

## 13. CachyOS Kernel Optimizations

### What CachyOS Ships
- **LTO (Link Time Optimization)**: Clang Thin LTO enabled by default
- **AutoFDO**: Profile-guided optimization using runtime data
- **Propeller**: Intel CPU optimization (not applicable to NVIDIA)
- **MGLRU patches**: Enhanced memory management
- **Distributed ThinLTO**: Parallel LTO compilation

### For LLM Inference
- CachyOS kernel is already optimized for performance
- No additional patches needed for llama.cpp
- The kernel's memory management improvements help with model loading

---

## 14. Key Models That Fit on GTX 1650 4GB

| Model | Quant | VRAM | Speed | Quality |
|---|---|---|---|---|
| Ornith-1.5-9B | IQ2_XXS | ~2.8GB | 20.0 tok/s | Good |
| Qwen3 4B | Q4_K_M | ~2.5GB | ~160 tok/s | Very good |
| Phi-3.5-mini 3.8B | Q3_K_S | ~3.9GB | ~255 tok/s | Good |
| Llama 3.2 3B | Q4_K_M | ~2.0GB | ~160 tok/s | Very good |
| Qwen3 1.7B | Q4_K_M | ~1.1GB | ~200 tok/s | Good |
| IBM Granite 3.1 MoE 1B | Q4_K_M | ~822MB | ~300 tok/s | Decent |

---

## 15. What Doesn't Work on 4GB VRAM (GTX 1650)

- ThunderKittens: needs SM80+ Tensor Cores
- Edge0: Apple Silicon only
- INT8 tensor-core GEMMs: no tensor cores on Turing
- 7B Q4 models without CPU offload (~5GB needed)
- DFlash1/Multi-step speculative decoding: drafter + target exceeds 4GB
- MTP speculative decoding: MTP drafter 5.78GB exceeds 4GB
- Long context (>8K) without KV cache quantization
- Multi-model concurrent serving

**Vulkan Backend Test Results (GTX 1650 4GB) — DEFINITIVE**
- Vulkan with 1 layer on GPU: **3.0 tok/s gen**, 3.8 tok/s prompt
- CUDA with 99 layers on GPU: **20.3 tok/s gen**, 63.6 tok/s prompt
- Vulkan is **6-7× SLOWER** than CUDA on 4GB VRAM
- Vulkan can only offload **1 layer** on 4GB VRAM vs CUDA's 99 layers
- Vulkan VRAM usage is pathological: **3.2GB for 1 layer** vs 2.8GB for 99 layers with CUDA
- `vkAllocateMemory` fails with `ErrorOutOfDeviceMemory` even with ~3GB free — contiguous memory fragmentation on Turing SM75
- **Verdict: Vulkan backend NOT viable on 4GB VRAM cards. Use CUDA.**

**SM75 Build Test (GTX 1650 4GB VRAM)**
- SM75 build with 1 layer on GPU: **2.98 tok/s gen**, 4.10 tok/s prompt
- CUDA build with 99 layers on GPU: **20.0 tok/s gen**, 50.8 tok/s prompt
- SM75 build is **6-7× SLOWER** than CUDA on 4GB VRAM
- SM75 build can only fit 1 layer in 4GB VRAM vs CUDA's 99 layers
- SM75 build tries to allocate 2.2GB for the model, exceeding available VRAM
- SM75 build OOM at 50 layers (same as Vulkan)
- **Verdict: SM75-specific build NOT beneficial on 4GB VRAM cards. Generic CUDA build is better.**

## Speculative Decoding Research (COMPLETED)
- Subagent: deleg_b02c4934 ✅
- Deliverable: /home/tov/speculative-decoding-research.md ✅
- Key finding: DFlash1 is BEST CANDIDATE for Ornith on 4GB VRAM
- DFlash2: NOT available for Ornith (only Qwen3.8-27B)
- MTP: retry with p-min=0.0, no separate drafter needed
- Ngram-cache: zero VRAM cost, always-on baseline
- Expected speedup: 1.3-1.6x with DFlash1 (20→26-32 tok/s)
- ACTUAL RESULT: DFlash1 2.7 tok/s, Ngram-cache 13.8 tok/s — BOTH SLOWER than baseline 20.0 tok/s
- **Verdict: Speculative decoding NOT beneficial on 4GB VRAM for Ornith-1.5-9B**

---

## 16. GGUF Quantization Advances (2025-2026)

### Beyond IQ2_XXS — What Actually Works on 4GB
- **IQ2_S** (2.5 bpw): Better quality-per-bit than IQ2_XXS; requires importance matrix (`llama-quantize --imatrix`). Use for models >13B where IQ2_XXS is too lossy.
- **IQ3_XXS** (3.06 bpw): Sweet spot for aggressive quantization — matches BF16 on AIME25 at 10.1GB for Qwen3.8-27B (GSQ-RCO). DASLab gradient-based tensor-level allocation beats fixed recipes.
- **GSQ-RCO** (ISTA-DASLab): Gradient-based precision allocation per tensor. IQ2_XS at 2.5bpw/8.4GB *outperforms* BF16 on zero-shot tasks for Qwen3.8-27B — first quant to beat FP16 on some benchmarks.
- **Unsloth Dynamic v2.0**: Per-tensor bit-width selection (Q4_NL/Q5.1/Q5.0/Q4.1/Q4_0) guided by KL-divergence. Claims ≤2% perplexity increase at 70% size reduction.
- **QKV-Core**: Adaptive hybrid quantization for 4GB GPUs — surgical block alignment + Numba acceleration. Targets GTX 1050/1650 class.

### Importance Matrix (imatrix)
- Per-tensor scaling based on activation magnitudes
- Improves IQ2_S/IQ3_XXS quality by 2-5% on benchmarks
- Command: `llama-quantize --imatrix imatrix.dat model.gguf model-imatrix.gguf`

---

## 17. CPU Offloading & Memory Mapping

### Key Techniques
- **--n-cpu-moe / -ngl**: Precise layer split. Build prints VRAM per layer; find the max that fits. Don't guess.
- **--mmap + --no-mmap**: mmap lets OS page model weights from disk; keep ON unless RAM is scarce.
- **DiskLLM**: KV cache on SSD via mmap — 23x RAM reduction for Qwen2.5-7B at 65K context (258MB vs 6GB). Requires NVMe.
- **llm-fit** (LD_PRELOAD shim): Intercepts cudaMalloc, redirects to cudaMallocManaged, inflates reported free VRAM. Measured 27→31/33 layers on RTX 3070 Laptop, 13→22 tok/s. Cap overflow at 1024MB (`LLM_FIT_MAX_OVERFLOW_MB`).
- **HotPin** (LozzKappa): For MoE models bigger than RAM — `mlock` top-K expert weights so OS can't evict them. 45.5% speedup on gpt-oss:120B with 19GB RAM. Zero quality loss (bit-identical).

### Rule
Usable VRAM on 4GB card ≈ 3.0-3.4GB after desktop + CUDA context. Plan for 70-75% of stated VRAM.

---

## 18. Vulkan Backend for Old NVIDIA (Detailed)

### Turing (SM75) Specific
- Vulkan on NVIDIA < Turing (pre-sm_75) has no tensor cores — cooperative matrix (coopmat) is the path
- On RTX 20xx (sm_75), coopmat2 landed in drivers R575+
- `GGML_VK_DISABLE_COOPMAT2=1` causes 30% throughput drop on RTX 4070 — keep enabled
- `GGML_VK_PREFER_HOST_MEMORY=1` adds 10-15% token/s on some configs
- For old NVIDIA cards, Vulkan can *outperform* CUDA (Quadro P620 issue #15955) — build from master, not distro packages

### Build Vulkan Backend
```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
cmake -B build -DGGML_VULKAN=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

---

## 19. CUDA Tricks for Turing SM75 (Detailed)

### Build for sm_75
```bash
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="75" \
  -DCMAKE_BUILD_TYPE=Release -DLLAMA_CURL=ON
cmake --build build -j
```

### Flags That Matter
- `-ngl 99`: Offload all layers; llama.cpp clamps to what fits
- `-fa on`: Flash attention — reduces compute buffer, enables KV cache quantization. 4.8% gain on Gemma 4 with no tensor cores
- `--cache-type-k q8_0 --cache-type-v q8_0`: KV cache quantization (8-bit), only works with flash attention
- `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1`: **Avoid on PCIe** — page-fault storms on Turing
- `--override-tensor 'blk.*.ffn_exps.*:CPU'`: For MoE, keep routed experts on CPU, attention in VRAM
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`: If using PyTorch path, reduces fragmentation OOM

### GTX 1650 Specific
- No tensor cores (int8 dot product only)
- Flash attention helps because it avoids materializing full attention matrix — critical when no tensor cores to hide dequant latency

---

## 20. MoE Expert Offloading (Detailed)

### VRAM-Saver Strategy (llama.cpp flags)
- `-ncmoe 999 -fa on -ctk q4_0 -ctv q4_0 -lm none`
- 75% VRAM reduction on 35B MoE model (15.7GB → 3.9GB on 16GB GPU)
- Attention stays on GPU, experts in system RAM
- 9× faster prompt processing vs brute-force GPU loading

### MoBiLE (Research Paper)
- Dynamically transfers only activated experts to GPU, offloads rest to CPU
- No training required, 1.6-1.72× speedup on consumer hardware (RTX 4080 16GB)

### FATE Expert Caching
- GPU-resident expert slot buffer with LRU eviction + cross-layer prefetching
- 99.94% cache hit rate at steady state

### Practical Recommendation for GTX 1650 4GB
- **Best bet**: Qwen3 4B dense at Q4_K_M (~2.5GB) — full GPU speed, good quality
- **MoE option**: Granite 3.1 MoE 3B (~2GB) or Qwen3 1.7B — MoE overhead not worth it at this VRAM tier due to PCIe bottleneck
- If using MoE: small MoE fully resident in GPU, or apply expert offloading with --cpu-moe — expect PCIe bottleneck on 4GB card
