# MoE Inference Engines — GitHub Scout Report
**Date:** 2026-09-18 | **Excluded:** llama.cpp, colibri, mesh-llm, FreeToken, Hermes MoA

---

## 1. Pure C/C++ Engines (Zero-Dependency, Single-File)

| # | Engine | Repo | Stars | License | VRAM/RAM | Speed | Key Innovation |
|---|--------|------|-------|---------|----------|-------|----------------|
| 1 | **kimi-k3-in-c** | FareedKhan-dev/kimi-k3-in-c | ~8k | Apache-2.0 | 8.2 GB RAM, CPU only | — | 2.78T param Kimi K3 on 8GB RAM CPU; custom mxFP4 quant, linear attention, AVX2 |
| 2 | **glm-5.2-in-c** | FareedKhan-dev/glm-5.2-in-c | ~65 | Apache-2.0 | 16 GB RAM, CPU optional CUDA | — | 744B GLM on 16GB RAM; pure C, one file (~3,900 LOC), int4 quant + expert streaming |
| 3 | **hummingbird** | prayangshuuu/hummingbird | ~57 | Apache-2.0 | SSD+RAM+VRAM hierarchy | — | Generalized Colibri-inspired modular runtime; C17, adapter-based architecture |
| 4 | **bitnet.c** | artalis-io/bitnet.c | ~29 | MIT | CPU + optional Metal/WebGPU | — | C11, Flash MoE (pread+LRU), TurboQuant 3-bit KV, 20+ GGUF formats, WASM compile |
| 5 | **Qwen_MOE_C** | h9-tec/Qwen_MOE_C | — | — | ~32-34 GB RAM | — | Pure C Qwen3 MoE, mmap weights, OpenMP, SIMD; single-file ~400-line stepping stone |

## 2. Expert / SSD Streaming Engines

| # | Engine | Repo | Stars | License | VRAM/RAM | Speed | Key Innovation |
|---|--------|------|-------|---------|----------|-------|----------------|
| 6 | **Flash-MoE** | danveloper/flash-moe | — | — | 48 GB Mac RAM, streams 209GB from SSD | 4.4+ tok/s | Pure C/Metal; Qwen3.5-397B-A17B on laptop; "trust the OS" page cache, no custom cache |
| 7 | **Flash-MoE Vulkan** | fluxism/flash-moe-vulkan | ~4 | MIT | ~6GB RAM, Vulkan GPU, SSD | — | Linux/Vulkan port of Flash-MoE; io_uring + GLSL compute shaders; 397B on AMD mini PC |
| 8 | **moe-stream** | GOBA-AI-Labs/moe-stream | ~2 | Apache-2.0 | 24 GB Mac (M4 Pro) | ~2.1 tok/s (80B SSD) | 3-mode auto-select (GPU Resident/Hybrid/SSD); Metal MXFP4 + Q4 quant matmul |
| 9 | **expert-stream** | Heman10x-NGU/expert-stream | ~1 | — | 16 GB RAM, no GPU needed | 2 tok/s | DeepSeek V4-Flash 284B on 16GB Windows laptop; mmap + FILE_FLAG_NO_BUFFERING |
| 10 | **MoE-Direct** | tmxkzm1925-max/MoE-Direct | ~11 | MIT | 32 GB RAM + RTX 5080 | 5.6 tok/s (122B) | Byte-preserving NVMe streaming; virtual repack (no double disk cost); CUDA |
| 11 | **MnemoCUDA** | AstrolexisAI/MnemoCUDA | ~1 | AGPL-3.0 | 12-46 GB VRAM cache + NVMe | ~14 tok/s warm | Multi-level VRAM cache (LRU+heat-pin), prefetch overlap, pipeline parallel across GPUs |
| 12 | **streamlx** | srcterm/streamlx | ~4 | MIT | RAM budget + NVMe | 19 tok/s (warm-started) | MLX-based Apple Silicon; fetch/compute overlapping; pool-backed expert wrapper |
| 13 | **tinyserve** | e1n00r/tinyserve | ~22 | MIT | 8 GB VRAM | 30 tok/s (20B MoE) | Zero-copy mmap expert store, ggml CUDA MMVQ, StreamingLLM for flat throughput |

## 3. Vulkan / CPU-Only Backends (Portable GPU)

| # | Engine | Repo | Stars | License | VRAM/RAM | Speed | Key Innovation |
|---|--------|------|-------|---------|----------|-------|----------------|
| 14 | **Kortex** | Vage91/Kortex | — | — | 20 GB GPU + NVMe | 160 tok/s (30B MoE) | Rust+wgpu; out-of-core streaming; ring-buffer VRAM slots; faster than llama.cpp on MoE |
| 15 | **infr** | kryptic-sh/infr | ~25 | MIT | Vulkan GPU + CPU offload | — | Pure Rust, Vulkan-first; INFR_NCMOE=N for MoE expert CPU offload; Q2_0 on GPU |
| 16 | **VulkanForge** | maeddesg/vulkanforge | ~22 | GPL-3.0 | AMD RDNA4 VRAM | — | First native FP8 WMMA over Vulkan on consumer AMD; 14MB static binary |
| 17 | **shimmy** | Michael-A-Kuykendall/shimmy | ~5.9k | Apache-2.0 | WebGPU (Vulkan/D3D12/Metal) | — | Pure Rust, zero-config; Airframe WGSL engine; MoE roadmap (not shipped yet) |
| 18 | **qwen-kernel** | ryanmurf/qwen-kernel | ~14 | MIT | Vulkan (RDNA3) | Parity w/ llama.cpp | Hand-written Vulkan kernels for every GGUF format; pre-recorded command buffers |
| 19 | **FerrisRes** | shift/FerrisRes | — | — | Vulkan/Metal/DX12 | — | Rust-native; Block AttnRes linear-time attention; MoE routing + 1.58-bit ternary |
| 20 | **meganeura** | kvark/meganeura | ~41 | MIT | Vulkan/Metal | — | Blade-graphics backend; portable training+inference; MoE not primary focus |

## 4. C++ Heterogeneous Runtimes

| # | Engine | Repo | Stars | License | VRAM/RAM | Speed | Key Innovation |
|---|--------|------|-------|---------|----------|-------|----------------|
| 21 | **ncnn-MoE-Runtime** | crafcat7/ncnn-MoE-Runtime | ~32 | BSD-2-Clause | CPU+Vulkan+memory+storage | — | Tencent/ncnn based; MoeIR intermediate representation; MXFP4 expert calibration |

---

## Key Themes Across Engines

1. **SSD Expert Streaming** is the dominant pattern for consumer hardware: keep ~5-10GB resident (attention/norms), stream 100-400GB of expert weights on demand. Hit rates of 60-90% achievable with LRU caches.

2. **Vulkan as the universal GPU backend**: Most new engines (Kortex, infr, VulkanForge, Flash-MoE Vulkan, qwen-kernel) use Vulkan via wgpu/ash/Vulkan SDK — no CUDA required.

3. **Pure C/C++ with zero dependencies**: Colibri-inspired single-file engines (kimi-k3-in-c, glm-5.2-in-c, bitnet.c, WARP) target embedded/air-gapped deployment.

4. **Memory hierarchy as first-class design**: The innovation is treating VRAM/RAM/NVMe as one placement problem with learned caches, not a constraint to work around.

## Gaps / Caution Flags

- Many repos are early-stage (WIP, pre-1.0) with minimal testing.
- Star counts may be inflated by trending/community hype (kimi-k3-in-c at 8k needs verification).
- Some "MoE" support is limited to specific model families (Qwen3, DeepSeek V4).
- No benchmarking standardization — results are model-specific and hardware-specific.
- Several repos (WARP, rlx-wgpu) had API issues fetching star counts; verify manually.
