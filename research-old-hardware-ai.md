# Linux & OmarchyOS Community Techniques for Old Hardware AI/LLM Inference

Research compiled September 2026 from Phoronix, LWN.net, GitHub, Reddit r/LocalLLaMA, Omarchy/Arch forums, CachyOS wiki, and community blogs.

---

## 1. Kernel-Level Optimizations

### Lazy Unmap Flush (LUF) — Linux 2025
- **What**: Defers TLB flushes until folios are unmapped and freed, reducing TLB shootdown interrupts by ~97%
- **Impact**: ~4.5% faster Llama.cpp inference; tested with 140GB memory workloads for stability
- **Status**: RFC patches on LKML; not yet mainline
- **Source**: Phoronix, Feb 2025

### TLB Flush Batching During Reclaim
- **What**: Batches TLB flushes for dirty folios in vmscan path — one IPI per PMD instead of per page (512x fewer IPIs)
- **Impact**: 26.9% throughput improvement on multi-core systems (Threadripper benchmark)
- **Author**: Tencent engineer Zhang Peng, March 2026
- **Source**: Phoronix, Mar 2026

### MGLRU (Multi-Gen LRU) Tuning
- CachyOS ships MGLRU enhancements: LRU-gen working set protection, compaction/watermark tweaks, hugepage reclaim
- Linux 7.3 merged patch keeps executable file folios mapped after first use — reduces system time from 9248s → 7861s on 32-core Arm server with 2GB memory limit
- **Community trick**: `/sys/kernel/mm/lru_gen/enable` = 1; `min_ttl_ms` to protect working set
- **Source**: Phoronix Aug 2026, CachyOS wiki

### AutoFDO + Propeller Kernel Profiling
- CachyOS default kernel uses AutoFDO + Propeller post-link optimization — ~10% throughput improvement
- Stackable: AutoFDO → ThinLTO → Propeller
- **Community trick**: `scripts/perf2event` to convert perf data to AutoFDO profile; rebuild kernel with profiling data
- **Source**: CachyOS blog, Feb 2025; LLVM Discourse

### Custom Kernel Distros for AI Hardware
- **CachyOS**: x86-64-v3/v4 builds, -O3, LTO, BORE scheduler, MGLRU, 1000Hz tickrate
- **Omarchy**: Custom `linux-omarchy` kernel (based on 7.2), Panther Lake `linux-ptl` with ~20 backports; idle as low as 2-3W on XPS
- **Source**: CachyOS wiki, Omarchy v3.5/v3.6/v4.0 release notes

---

## 2. OmarchyOS-Specific AI Techniques

### Omarchy AI Performance Engine (Discussion #10518)
- Community proposal for `omarchy-ai detect` — hardware-aware auto-optimization
- Detects: CPU ISA, RAM speed, GPU, Vulkan/SYCL/CUDA/ROCm backends
- Auto-generates optimization profile: backend selection, thread count, GPU layers, batch/ubatch sizes
- **Status**: Open discussion, not yet implemented

### DHH's AMD Strix Halo Setup (Goran Pavlovic Blog)
- **Omarchy + Linux Zen kernel + llama.cpp** serving Qwen3 Coder Next with speculative decoding
- **Key trick**: Set `amdgpu.gttsize=100000` in `limine.conf` to give iGPU access to 100GB of unified memory
- BIOS: dedicated VRAM set to minimum 512MB (carved from system RAM); GTT memory gets the rest
- 100GB allocated for iGPU AI models; 28GB left for OS
- **Command**: `cmdline: ... ttm.pages_limit = 26214400 amdgpu.gttsize = 100000`

### Omarchy v4.0 (Quattro) AI Features
- Agentic OS: AI crash diagnosis via `diagnose-crash` skill
- Lazy-install coding agents (Claude Code, Codex, OpenCode, Gemini, etc.)
- Swap tuned on zram instead of hibernation swapfile
- Model-usage bar widget tracks token usage by day/model
- **Source**: Omarchy v4.0.0 release notes

---

## 3. llama.cpp Community Tricks for Old Hardware

### CPU-Only Inference on 2-Core/8GB DDR2 (Reddit r/LocalLLaMA #21136)
- **OS**: Linux Mint 22.3 MATE, kernel 6.14, nomodeset boot
- **Build**: `cmake -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DLLAMA_BUILD_SERVER=ON`
- **Critical flags**: `--mlock` (locks model in RAM), `-t 2` (match physical cores), `-ngl 0` (no GPU), `--ctx-size 2048`
- **Model**: Qwen3.5-4B Q4_K_M → ~2 tokens/sec on 2-core Core2 Duo
- **Trick**: Set `vm.swappiness=10` to prevent kernel from swapping model pages; set `memlock unlimited` in limits.conf

### Memory-Mapped Model Loading
- llama.cpp uses mmap for model weights — OS pages in only active layers
- Allows running models 10x larger than RAM (70B model on 16GB machine)
- **Trick**: Use SSD for model storage; sequential page faults are fast

### KV Cache Quantization Symmetry
- **Critical finding**: Mismatched KV precision (f16 key + q4_0 value) forces format conversion mid-execution, tanking prefill to 33.4 tok/s
- Symmetric quantization (q4_0 for both key and value) → 20.3 tok/s, 75% VRAM savings
- **Community trick**: Always use `-ctk q4_0 -ctv q4_0` together

### GPU Layer Offload Cliff
- Offloading 4-8 layers on a 6GB GPU *hurts* performance (PCIe transfer overhead)
- Offload all layers or almost none — performance only scales past 50% offload
- **Community trick**: Benchmark with `llama-bench` at your exact `-ngl` value

### Batch/Micro-Batch Tuning
- Increasing micro-batch (-ub) from 128→512: up to +85% prefill speed
- Decode speed unaffected by batch size
- **Community trick**: `-b 512 -ub 128` for RAG pipelines (prefill-heavy)

### CPU Thread Optimization on Asymmetric CPUs
- Match `-t` to physical core count, NOT logical threads
- Intel i5-13450HX (6P+4E): prefill peaks at 10 threads; 12 threads drops performance (P-cores wait for E-cores)
- **Community trick**: `llama-bench -t 4,6,8,10,12` to find sweet spot

### AVX/AVX2/AMX Compilation
- Build from source: auto-detects CPU ISA, yields 10-20% over prebuilt binaries
- **Community trick**: `make LLAMA_METAL=0 LLAMA_CUDA=0 -j$(nproc)` with `CMAKE_C_FLAGS="-march=native -O3"`
- KleidiAI integration for ARM NEON/SVE matrix multiply

### Vulkan Backend for Old GPUs
- llama.cpp b9932: conditional Flash Attention masking for AMD GCN — disables for head_size ≤ 256, re-enables above
- Intel Arc A750: Vulkan backend works without ReBAR; SYCL failed on old CPUs (i7-6950X too old for IPEX-LLM)
- **Community trick**: Use Vulkan over SYCL on older Intel integrated graphics

---

## 4. ZRAM/ZSWAP for Memory-Constrained Systems

### ZRAM (Compressed RAM Swap)
- Creates compressed block device entirely in RAM — zero disk I/O
- **Trick**: Size = 50-100% of RAM; zstd for modern CPUs, lz4 for old/low-power
- **Community config** (`/etc/sysctl.d/99-vm-zram.conf`):
  ```
  vm.swappiness = 180
  vm.watermark_boost_factor = 0
  vm.watermark_scale_factor = 125
  vm.page-cluster = 0
  ```
- Pop!_OS uses these values; ~3:1 compression ratio typical
- **Source**: Arch Wiki, fosslinux.com, Reddit r/LocalLLaMA

### ZSWAP (Compressed Cache Before Disk Swap)
- Better for servers with NVMe: caches compressed pages, evicts cold ones to disk
- `zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=20` on kernel cmdline
- **Source**: ZDNET, computingforgeeks.com

---

## 5. Community Patches & Forks

### llama.cpp Hacks
- `yshui/llama.cpp-hacks`: Face GGUF editor, unofficial patches
- Community cherry-picks for Intel Arc B70: 11 commits fixing MoE slot-init SEGV, Q8_0 reorder crash, missing BF16 GET_ROWS, Xe2 Vulkan warptile
- **Result**: 2-7x speedup on Intel Arc Pro B70; 59.6 tok/s vs stock hang

### LLM-Tweaks (Bingi-Man)
- Comprehensive GitHub guide covering: quantization, GGUF/llama.cpp, model selection, compilation, GPU acceleration
- Specific recommendations for 8GB GPU: Q4_K_M GGUF, partial offload, ngl=20-30

### CachyOS Kernel Patches
- 15 upstreamed topic branches: MGLRU, BBR3, CACHY (scheduler), cgroup-vram (DMEM), AMD P-State, T2 MacBook
- ADIOS v3.2 I/O scheduler, BFQ enhancements
- NVIDIA proprietary driver modules with patches
- **Source**: CachyOS github, wiki

---

## 6. Old Hardware AI Stack Recommendations (Community Consensus)

### Minimal Setup (2-core, 8GB RAM, no GPU)
- OS: MX Linux or AntiX Linux (lowest RAM footprint)
- Kernel: 6.14+ with nomodeset boot
- Engine: llama.cpp CPU-only, Q4_K_M quant, --mlock, swappiness=10
- Model: ≤4B params (Qwen3.5-4B, MX4)

### Mid-Range (8GB RAM, old GPU Pascal/Volta)
- OS: Arch/CachyOS with x86-64-v3 kernel, BORE scheduler
- Engine: llama.cpp with Vulkan backend
- ZRAM: 50% RAM, zstd, swappiness=180
- Model: 7-8B Q4_K_M, full GPU offload or hybrid

### High-Attempt (16GB RAM, iGPU only — Omarchy Strix Halo style)
- `amdgpu.gttsize=100000` in bootloader
- 100GB iGPU memory pool, 512MB dedicated VRAM minimum
- llama.cpp Vulkan/SYCL, large MoE models
- Model: 30-35B MoE (Qwen3.6-35B-A3B)

---

## 7. Undocumented / Cutting-Edge Tricks

1. **NUMA distribute on dual-socket**: `llama-bench --numa distribute -t` — 80% token gen improvement on dual Epyc Turin (but can hurt MoE models)
2. **Drop caches before load**: `echo 3 > /proc/sys/vm/drop_caches` then load model, then generate — keeps model in memory cache
3. **Limine bootloader**: Omarchy's bootloader supports `amdgpu.gttsize`, `zswap.enabled`, custom kernel params in `limine.conf`
4. **CACHE-SIZE-AWARE prefetch**: llama.cpp PR #295 demonstrated ~25% improvement via prefetch + CPU pinning + unroll on i7-7700k (memory-bound optimization)
5. **Speculative decoding on old hardware**: DFlash draft model on DGX Spark — 2.0-2.6x token generation increase, but requires specific vLLM patches
6. **FP8 on old NVIDIA**: Not applicable — FP8 needs Ada/Blackwell (RTX 40-series+). INT8 is the floor for old cards.
