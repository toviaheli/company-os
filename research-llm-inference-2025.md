# 2025-2026 LLM Inference Optimization for Old Hardware (GTX 1650 4GB, Turing SM75)

## Research Date: September 2026

---

## 1. PagedAttention Variants for Low VRAM

### PagedAttention (vLLM, 2023/ongoing 2025)
- Breaks KV cache into fixed-size pages with block-table indirection (OS virtual memory analogy)
- Eliminates external fragmentation; minimal internal fragmentation only
- Enables dynamic allocation/reuse without pre-allocating max contiguous memory
- **Relevance to SM75**: vLLM's PagedAttention is framework-level, not GPU-arch-specific — works on any CUDA GPU

### vAttention (Microsoft Research, ASPLOS 2025)
- Alternative to PagedAttention: retains KV cache in contiguous virtual memory
- Uses OS-level demand paging for on-demand physical allocation
- **Key advantage**: No attention kernel rewrite needed; works with unchanged implementations
- **Performance**: 1.97× faster token generation than vLLM, 3.92× faster prefill than FlashAttention-PagedAttention variant
- **SM75 potential**: Since it doesn't require kernel changes, it's a drop-in option for any framework

### PagedAttention + FlexAttention (IBM, June 2025)
- Fused integration within IBM Foundation Model Stack
- On NVIDIA L4 (24GB): near-linear latency scaling from 128→2048 tokens
- Minimal incremental memory overhead from paged attention at shorter sequences

---

## 2. KV Cache Compression Techniques

### KVTC (Transform Coding, Nov 2024 arxiv)
- PCA-based feature decorrelation + adaptive quantization + entropy coding
- **Up to 20× compression** while maintaining reasoning accuracy
- Brief calibration needed; leaves model parameters unchanged

### KVComp (Sept 2024 arxiv)
- Lossy compression with error-controlled quantization + GPU entropy encoding
- **47% avg, up to 83% memory reduction** vs. existing methods
- Decompressed data consumed in situ within GPU shared memory

### RocketKV (Feb 2025 arxiv)
- Two-stage: coarse permanent eviction + fine top-k sparse attention
- **7× speedup, 32.6% peak memory reduction** on A100 decode phase
- Training-free; integrates with existing attention implementations

### Mustafar (2025 arxiv)
- Unstructured sparsity for KV cache pruning: up to 70% sparsity
- Per-token magnitude-based pruning for K and V caches
- Bitmap-based sparse format + custom attention kernel
- **Up to 2.23× throughput** vs. dense inference
- SM75-compatible: no Tensor Core requirement

### TailorKV (May 2025 arxiv)
- Hybrid: quantizes "quantization-friendly" layers + offloads "sparsity-friendly" layers to CPU
- 1-bit per float quantization on compatible layers
- Hardware-friendly implementation

---

## 3. Dynamic Batch Size & Continuous Batching

### Continuous Batching (Orca, OSDI 2022 → production 2023-2026)
- Iteration-level scheduling: make scheduling decision every token step, not per batch
- **36.9× throughput** improvement over FasterTransformer (Orca paper)
- **4-8× real-world throughput** on mixed short/long workloads
- **GPU utilization**: 55% → 95% on same hardware

### Key Implementation Details
| Engine | Term | Key Knobs |
|--------|------|-----------|
| vLLM | Continuous batching | `--max-num-seqs`, `--max-num-batched-tokens`, `--enable-chunked-prefill` |
| TensorRT-LLM | In-flight batching | `inflight_batching` build flag |
| SGLang | Continuous batching | + RadixAttention for cross-request KV reuse |
| LMDeploy | Persistent batching | Same mechanism |

### Chunked Prefill (Sarathi-Serve → vLLM default)
- Splits long prompts into 512-2048 token chunks interleaved with decode
- Trade-off: decode latency vs. prefill throughput
- **Critical for SM75**: prevents one long prefill from stalling all decoding sequences

---

## 4. Custom Attention Kernels for Turing SM75

### FlashAttention on SM75 (John Scheuer, github.com/JohnScheuer/flash-attention-sm75)
- Full CUDA FlashAttention v1 forward pass for SM75 (RTX 2070/2080, T4)
- Uses Tensor Cores (WMMA) for QK^T and PV computation
- **O(N) memory** vs O(N²): enables longer sequences
- End-to-end throughput: **+27%** on Qwen2-0.5B (31→40 tok/s)
- **Caveats**: FP16 only, head_dim 64/128, no decode KV cache acceleration
- SM75 shared memory limit: 48 KB per block

### llama.cpp FlashAttention Updates (2025-2026)
- **PR #13435**: CUDA FA kernel optimized for Deepseek + Turing support
  - Batch sizes tunable per compute capability
  - Turing gets 64 KiB SRAM per SM (vs 99+ KiB on Ampere)
  - KV cache size reducible by ~47% when V cache deduplicated into K
- **PR #17505**: Generalized (mma) FA, added Volta/SM7x support
  - MMA kernel replaces WMMA for SM75
  - Prefill performance boost on V100/SM7x: ~1.1-1.7×
- **PR #16492**: Tile FA kernel improvements
  - Better small batch size performance
  - GQA optimizations reduce I/O

### Split-KV Attention (from llama.cpp old-hardware skill)
- **1.5-3× speedup on SM75** specifically
- Reduces memory bandwidth by splitting K and V computation

### Embedding Requantization
- INT8 group-128 requantization: **2.6GB savings**
- Applied to embedding layers during inference

---

## 5. Memory-Efficient Attention Techniques

### FlashAttention-2/3
- Not SM75-compatible (requires SM80+ Tensor Cores)
- Use SM75-specific FlashAttention v1 fork instead

### Memory-Efficient Attention (xFormers, etc.)
- Limited SM75 support in maintained forks
- PyTorch SDPA efficient backend works on SM75 but is closed-source

### Online Softmax (FlashAttention core)
- Running max/sum instead of materializing N×N matrix
- O(N) memory complexity
- Already implemented in SM75 FlashAttention fork above

---

## 6. New Frameworks & Approaches for Old GPU Inference

### LvLLM (v2.3.6, July 2026)
- vLLM fork with NUMA-aware CPU+GPU hybrid inference
- **Explicit SM75 GPU prefill support**: `sm75` dtype float16 prefill
- ModelOpt W4A16 NVFP4 quantization types
- Hybrid decoding: CPU-GPU split for models exceeding VRAM
- Requirements: x86 CPU with AVX2+, NVIDIA SM75+

### FATE-llama.cpp (MoE Expert Offloading)
- Cross-layer expert prefetching with shallow-favoring cache
- **3-5× speedup** on MoE models with offloaded experts
- 99% expert hit rate via gated caching strategy
- Works on consumer GPUs with limited VRAM

### Vulkan Backend (llama.cpp)
- **37% faster than CUDA on GTX 1060**, similar gains expected on GTX 1650
- Install: `yay -S llama.cpp-vulkan`
- Alternative to CUDA for Turing-generation GPUs

### Prompt Lookup Drafting
- Theoretical 381 tok/s speedup
- Requires 5-10GB (may not fit on 4GB alone)
- Best combined with other techniques

---

## 7. Models That Fit on 4GB VRAM (2025-2026)

| Model | Quant | VRAM | Speed | Notes |
|-------|-------|------|-------|-------|
| Ornith-1.5-9B | IQ2_XXS | ~2.8GB | 20.0 tok/s | Best speed/quality on 4GB |
| Qwen3 4B | Q4_K_M | ~2.5GB | ~160 tok/s | Strong general purpose |
| Phi-3.5-mini 3.8B | Q3_K_S | ~3.9GB | ~255 tok/s | Near-full VRAM |
| IBM Granite 3.1 MoE 1B | Q4_K_M | ~822MB | ~300 tok/s | Fastest, smallest |
| Qwen3 0.6B | Q4_K_M | ~0.5GB | — | Ultra-small, fast |
| Qwen3 1.7B | Q4_K_M | ~1.1GB | — | Good balance |

---

## 8. Recommended Optimization Stack for GTX 1650 4GB

### Tier 1: Quick Wins (no code changes)
1. **IQ2_XXS or IQ1_M quant** — 2-3GB, 20+ tok/s
2. **FlashAttention on** — saves VRAM, slight speed gain
3. **Vulkan backend** — 37% gain over CUDA expected
4. **n_gpu_layers=85** — prevents OOM

### Tier 2: Kernel-Level
5. **Split-KV Attention** — 1.5-3× speedup on SM75
6. **SM75 FlashAttention fork** — +27% end-to-end throughput
7. **Embedding requant int8 group-128** — 2.6GB savings

### Tier 3: Advanced
8. **LvLLM** — CPU+GPU hybrid for models exceeding VRAM
9. **FATE offloading** — for MoE models (3-5× speedup)
10. **KV cache compression** (Mustafar/RocketKV) — up to 70% KV cache reduction
11. **vAttention** — if using vLLM-based serving

### Key Tuning Flags (vLLM)
```
--max-num-seqs 256-1024
--max-num-batched-tokens 4096
--enable-chunked-prefill
--block-size 16
--gpu-memory-utilization 0.90
--kv-cache-dtype fp8 (if supported)
```

---

## Key Insights

1. **PagedAttention + Continuous Batching** are the foundation — every modern serving stack has them
2. **SM75-specific kernels matter**: FlashAttention v1 fork gives +27% end-to-end despite raw kernel being slower than SDPA
3. **Vulkan often beats CUDA on old GPUs** — 37% gain documented on GTX 1060, likely similar on 1650
4. **MoE models + expert offloading** are the best path to larger models on 4GB
5. **KV cache compression** (Mustafar, RocketKV) can reduce memory 30-70% with minimal accuracy loss
6. **vAttention** (Microsoft, ASPLOS 2025) is the most practical drop-in alternative to PagedAttention — no kernel changes needed
7. **LvLLM** is the most promising framework for CPU+GPU hybrid on SM75, with explicit Turing prefill support

---

## Sources
- PagedAttention paper (Kwon et al., 2023) + vLLM docs
- vAttention (Microsoft Research, ASPLOS 2025)
- PagedAttention + FlexAttention (IBM, arXiv 2506.07311)
- KVTC (arXiv 2511.01815)
- KVComp (arXiv 2509.00579)
- RocketKV (arXiv 2502.14051)
- Mustafar (arXiv, djhoo98)
- TailorKV (arXiv 2505.19586)
- Orca continuous batching (OSDI 2022)
- FlashAttention SM75 fork (JohnScheuer)
- llama.cpp PRs #13435, #17505, #16492
- LvLLM github (guqiong96)
- FATE-llama.cpp (arXiv 2502.12224)
- Hermes skills: cutting-edge-llm-optimization, llm-optimization-old-hardware
- packet.ai/continuous-batching, vllm.ai blog