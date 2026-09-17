# Cutting-Edge LLM Optimization Techniques for Old Hardware (GTX 1650 4GB VRAM)

**Research date:** 2026-09-16
**Target hardware:** NVIDIA GTX 1650 (Turing SM75, 4GB GDDR6, 192 GB/s bandwidth)
**Confidence:** Medium-High (based on web sources, papers, and community benchmarks)

---

## 1. Model Quantization — GGUF Format Advances

### The State of GGUF (2026)

The llama.cpp GGUF ecosystem is the dominant quantization framework for consumer hardware. Key findings:

**K-Quant family (superblock-based):** The modern K-quant formats (Q2_K through Q6_K) use superblocks with mixed-precision sub-blocks, significantly outperforming legacy formats (Q4_0, Q4_1, Q5_0, Q5_1, Q8_0) at the same nominal bit-width [arXiv:2601.14277].

**Unified evaluation (Jan 2026 paper):** A comprehensive study of Llama-3.1-8B-Instruct across all GGUF formats found:
- Q4_K_M remains the best quality/bit tradeoff for 4GB VRAM
- Q3_K_M fits 7B models in ~3.8GB but with noticeable quality loss
- Q5_K_M on 3B models gives near-original quality with ~2.5GB VRAM

**Recommended quantization tiers for 4GB VRAM:**
| Model | Quant | VRAM | Quality |
|-------|-------|------|---------|
| SmolLM2 1.7B | Q8_0 | ~2GB | Best possible |
| Qwen3 4B | Q4_K_M | ~2.5GB | Sweet spot |
| Phi-4-mini 3.8B | Q4_K_M | ~2.5GB | Best reasoning |
| Llama 3.2 3B | Q5_K_M | ~2.9GB | More headroom |
| 7B models | Q3_K_M | ~3.8GB | Borderline |

**imatrix (importance matrix):** Post-training calibration using a calibration dataset produces better low-bit quantization. Use `llama-imatrix` before quantizing to Q2_K/Q3_K for 10-15% quality recovery [llama.cpp docs].

**New in 2025-2026:** GGUF supports bf16 output type, FP8 quantization for multimodal encoders, and Q8_K quantization (8-bit K-quant) for models that just barely fit.

---

## 2. Kernel Fusion & SM75-Specific CUDA Optimizations

### Turing (SM75) Constraints

GTX 1650 = Turing SM7.5, no Tensor Cores, 192 GB/s memory bandwidth. This is the critical bottleneck — LLM inference is memory-bandwidth bound, and the GTX 1650 has 5-6x less bandwidth than an RTX 4090.

### Vulkan Backend vs CUDA on Old NVIDIA GPUs

Surprising finding: On GTX 1060 (SM61), the Vulkan backend in llama.cpp achieved **37% faster token generation** than CUDA for some models (granitehybrid 1B Q4_K: Vulkan 90.6 tok/s vs CUDA 61.7 tok/s at tg128) [GitHub issue #19817]. The Vulkan backend uses SPIR-V shaders with cooperative matrix optimizations that can outperform CUDA on older hardware where CUDA kernel launch overhead dominates.

**Vulkan backend advantages for SM75:**
- SPIR-V cooperative matrix (VK_KHR_cooperative_matrix) for hardware-accelerated matmul
- Flash Attention via SPIR-V shaders (flash_attn.comp, flash_attn_cm1.comp, flash_attn_cm2.comp)
- Cross-vendor compatibility with same code path
- Configurable tiling parameters (BLOCK_SIZE, BM, BN, BK) matched to warp size

**Vulkan limitations:**
- No FP16 buffer storage on Turing (fp16: 0 in device query) — uses workaround
- No BF16 support on SM75
- Prompt processing (prefill) significantly slower than CUDA on NVIDIA

### Kernel Fusion

**Megakernel approach:** Single persistent CUDA kernel for entire forward pass eliminates ~100 kernel launches per token. The Luce-Org megakernel for Qwen 3.5-0.8B fuses: DeltaNet recurrence via warp-cooperative state updates, full attention with online softmax, cooperative grid sync between layers, in-kernel KV cache updates [Luce-Org/lucebox].

**Key fusion targets for old GPUs:**
- Fused LayerNorm + linear projection
- Fused activation (gelu/silu) + gate multiplication for gated MLPs
- Fused attention (FlashAttention)
- Fused residual add + normalization
- torch.compile with inductor backend for automatic fusion

### FlashAttention on Old GPUs

FlashAttention reduces memory from O(n²) to O(n) for attention intermediates. On SM75:
- Supported in llama.cpp via `--flash-attn` flag (both CUDA and Vulkan)
- Primary benefit is memory reduction, not speed — on old GPUs with small models that fit in VRAM, speedup may be negligible
- Vulkan FlashAttention uses subgroup shuffles and shared memory reductions for cross-vendor compatibility [DeepWiki antirez/llama.cpp-deepseek-v4-flash]

---

## 3. CPU+GPU Hybrid Inference

### Real-World GTX 1650 4GB Data

A GTX 1650 Ti Mobile test with Ollama + Gemma 4 E2B showed [community benchmark]:
- CPU only: 17 tok/s, 3506ms eval latency
- GPU hybrid (35/36 layers): 39 tok/s, 1411ms eval latency
- **2.5x speedup** — but only 2.5x, not 10x, because the output layer stays on CPU

**The critical insight:** Hybrid inference is gated by the slower device. On 4GB cards, the GPU can hold most layers but the output embedding + LM head must remain on CPU. Every generated token round-trips through CPU for that final layer, making it the steady-state bottleneck.

### Layer-Wise Offloading Strategies

**AirLLM:** Enables 70B parameter inference on 4GB GPUs via aggressive layer-wise quantization and CPU-GPU streaming [aisignal.dev]. Uses memory-mapped file I/O treating SSD as extended RAM. Each layer transfer (~400MB for 70B) crosses PCIe every forward pass — speed is correlated with PCIe bandwidth, not GPU compute.

**ATSInfer (2026 paper):** Automated tensor-level placement for hybrid CPU-GPU inference on consumer devices [arXiv:2607.10183]:
- Static tensor placement + load-aware dynamic transfer
- Asynchronous CPU-GPU coordination
- Prioritizes tensors by latency reduction per byte of GPU memory
- **1.94x prefill improvement, 3.29x decode improvement** over llama.cpp under same VRAM budget
- 70% higher average GPU utilization during decode

**Pipelined Sharding (MLSys 2026):** Sub-layer-level model sharding with CPU offloading, pipelined copy-compute, prioritized tensor placement [arXiv:2604.26334]:
- TTFT up to 6.7x improvement, TPS up to 30x for LLMs
- Can run Qwen-235B (77GB) on 2GB VRAM
- Flexibly adapts to system conditions (CPU threads, PCIe bandwidth, VRAM budget)

---

## 4. Model Sharding Across Devices

### Practical Approaches for Single-GPU 4GB Systems

**llama.cpp tensor splitting:** `-tg` flag splits model across CPU + GPU. Not true sharding — more like layer offloading. Limited by PCIe bandwidth.

**Multi-GPU tensor parallelism:** vLLM supports tensor parallelism across GPUs via Megatron-LM algorithm. For 4GB single-GPU setups, this requires a second GPU (even an old GTX 1060 6GB helps).

**Pipeline parallelism:** vLLM supports pipeline parallel for models too large for single GPU. Requires multi-GPU or CPU offload.

**GGUF partial offload:** llama.cpp supports `-ngl` (num-gpu-layers) to control how many layers go on GPU. Optimal value depends on model size:
- 3B Q4_K_M: `-ngl 30` (all layers fit)
- 7B Q3_K_M: `-ngl 40-50` (partial offload, rest on CPU)
- Each additional GPU layer increases VRAM usage by model_size / n_layers

### The PCIe Bottleneck

On a GTX 1650 (PCIe 3.0 x16 = 16 GB/s theoretical, ~12 GB/s real), layer transfers for a 7B Q4 model (~3.5GB) take ~300ms per layer switch. This makes frequent CPU-GPU switching more expensive than keeping layers statically placed.

---

## 5. KV Cache Compression

### Cutting-Edge Methods (2025-2026)

**Eigen Attention (arXiv:2408.05646):** Performs attention in low-rank space via SVD on calibration data. Reduces KV cache by 40%, attention latency by 60%. Orthogonal to other compression methods — can be combined. Post-training, no fine-tuning needed.

**KV-Compress (arXiv:2410.00161):** Evicts contiguous KV blocks within PagedAttention framework:
- Variable-rate eviction across layers and heads (up to 8x compression)
- Compatible with vLLM
- Up to 5.18x throughput improvement
- Squared past attention metric for eviction decisions

**RocketKV (arXiv:2502.14051):** Two-stage KV cache compression:
- Stage 1: Coarse-grain permanent KV eviction (SnapKV)
- Stage 2: Fine-grain dynamic top-k sparse attention (hybrid sparse attention)
- Up to 400x compression, 3.7x speedup, 32.6% memory reduction
- Adaptive compression decomposition across stages

**ALISA (arXiv:2403.17312):** Sparsity-aware KV caching with three-phase dynamic scheduling:
- Sparse Window Attention (SWA) identifies important tokens
- Phase I: GPU caching (short sequences)
- Phase II: GPU-CPU caching (growing sequences)
- Phase III: Recomputation-caching (very long sequences)
- KV compression via INT8 quantization (50% reduction)
- Up to 3x throughput improvement over FlexGen

**LeoAM (arXiv:2506.20187):** Adaptive hierarchical GPU-CPU-Disk KV management:
- Adaptive chunk sizing based on attention sparsity
- Lightweight KV abstract for disk-based importance evaluation (13.25% of original data)
- 3.46x average inference latency speedup, up to 5.47x at large batch sizes
- Tree-structured KV management for precision

**CacheTune (arXiv:2605.24022):** Frequency-guided KV cache reuse:
- Frequency-domain analysis identifies critical KV pairs
- Selective recomputation of semantic-critical tokens
- 3.72x-4.86x TTFT speedup, 3.93x-6.21x throughput
- Works even with SSD/HDD offloading (2.34x-2.36x TTFT)

### Practical Implications for 4GB VRAM

With ~1-1.5GB left for KV cache after model weights:
- Standard FP16 KV: ~2-4K tokens context
- INT8 KV quantization: ~4-8K tokens
- Aggressive KV compression (Eigen/ALISA): ~8-16K tokens possible
- Disk offloading (LeoAM): virtually unlimited context, but slower

---

## 6. Prompt Caching

### Provider-Level Prompt Caching

OpenAI, Anthropic, and Google offer prompt caching at the API level — reusing KV tensors across requests with shared prefixes [arXiv:2601.06007]. Reduces cost and TTFT for repeated prompts.

### Local Prompt Caching (llama.cpp)

**llama.cpp prompt cache:** Reuses computed KV cache for repeated prompts. On RTX 2070 (SM75), FreeToken's prompt cache made repeated warm-long prompts **50.17% shorter** than competing implementations [pocketai-freetoken-sm75].

**How it works:**
- First request: compute and cache KV tensors for prompt prefix
- Subsequent requests with same prefix: load cached KV instead of recomputing
- Cache key is the prompt text hash
- Works with any model/quantization

**CacheTune (research):** Extends prompt caching beyond strict-prefix matching using frequency-domain analysis to identify reusable KV pairs even in non-prefix scenarios.

**RelayCaching (arXiv:2603.13289):** Reuses decoding-phase KV caches from previous agents in prefill phases — 80% KV cache reuse, up to 4.7x TTFT reduction in multi-agent pipelines.

### Practical Setup for GTX 1650

```bash
# llama.cpp with prompt cache
./llama-cli -m model-Q4_K_M.gguf -ngl 35 --prompt-cache-path ./cache --prompt-cache-retokenization 0
```

The cache persists across sessions, dramatically reducing TTFT for repeated system prompts.

---

## 7. Attention Optimization for Old GPUs

### FlashAttention Variants

- **FlashAttention-1:** Exact attention, O(n) memory, 2-4x faster for long sequences on A100
- **FlashAttention-2:** Better parallelism across sequence length
- **FlashAttention-3:** Hopper-only (SM90), not relevant for SM75

**For SM75 (GTX 1650):** FlashAttention is supported in llama.cpp but the benefit is primarily memory reduction, not speed. On old GPUs with small models fitting in VRAM, the speedup from memory bandwidth savings is modest because the bottleneck is compute, not memory.

### Sliding Window Attention

Fixed-size window attention — only attends to last W tokens:
- Reduces KV cache from O(n) to O(W)
- Implemented in llama.cpp via `--sliding-window` flag
- Quality loss depends on model architecture; works best with models trained with sliding window (Gemma 2, Phi-3)
- For 4GB VRAM: window size of 2048-4096 is practical

### Sparse Attention

**ALISA's SWA:** Dynamically identifies important tokens + static local window
- More accurate than fixed sliding window
- Sparsity increases with sequence length (95-97% for large models)

**Quest/SparQ:** Dynamic top-k attention — only computes attention for top-k tokens per query
- Training-free, uses token importance scores
- Can be combined with permanent eviction (RocketKV approach)

### Multi-Query / Grouped-Query Attention

MQA/GQA reduces KV cache by sharing key-value heads across query heads:
- MQA: 1 KV head for all query heads (8x KV cache reduction for 8-head models)
- GQA: Multiple KV heads (4x for GQA-4, 2x for GQA-2)
- Most modern models (Llama 3, Qwen 2.5, Gemma 2) use GQA — no extra cost to enable

### Low-Rank Attention (Eigen Attention)

Projects K,V into low-rank space via offline SVD:
- 40% KV cache reduction, 60% attention latency reduction
- Post-training, orthogonal to other methods
- Particularly effective for older models not trained with GQA

---

## 8. Community Techniques for 4GB VRAM

### The "Sweet Spot" Model Stack (2026)

Based on actual testing on 4GB GPUs (GTX 1650, RX 580, RX 6500 XT):

**Best models for 4GB VRAM:**
1. **Qwen3 4B Q4_K_M** — 2.5GB VRAM, best all-rounder for chat
2. **Phi-4-mini 3.8B Q4_K_M** — Best reasoning tasks
3. **Qwen3 4B Q5_K_M** — 2.9GB VRAM, higher quality if you can spare the memory
4. **SmolLM2 1.7B Q8_0** — 2GB VRAM, fastest possible, good for classification
5. **Llama 3.2 3B Q5_K_M** — 2.9GB VRAM, good multilingual

**What doesn't work:**
- 7B Q4_K_M: ~4.4GB — doesn't fit, borderline with Q3_K_M
- 7B models on 4GB: only viable with aggressive offloading and short context

### Practical Tips from the Community

**Vulkan backend on NVIDIA:** Surprisingly competitive on old GPUs. Test with:
```bash
llama-cli -m model.gguf --backend vulkan -ngl 35
```

**Context window reality check:** On 4GB VRAM with a 3B Q4_K_M model (~2GB), you have ~1-1.5GB left for KV cache = 2-4K tokens practical context. With Q8 KV cache quantization you can stretch to 6-8K tokens.

**Temperature under load:** GPU hybrid mode runs ~10°C cooler than CPU-only (the GPU shares the thermal load) [community benchmark].

**Ollama vs llama.cpp:** Ollama auto-splits layers but may leave the output layer on CPU. llama.cpp gives manual control via `-ngl`. For 4GB, explicit layer control in llama.cpp typically outperforms Ollama's auto-split.

**vLLM on 4GB:** Possible but problematic — vLLM's CUDA dependency chain is heavy, and paged attention overhead can exceed benefits at this VRAM level. Docker WSL2 adds further overhead. Community reports mixed success [Medium guide].

**AirLLM for 70B:** Technically works on 4GB (layer-wise offloading), but at ~20 seconds per token, it's more of a demo than a practical tool. 10-50x slower than llama.cpp with quantized models.

### Hardware-Specific Optimizations for GTX 1650

```bash
# Recommended llama.cpp build flags for SM75
cmake -B build -DGGML_CUDA=ON -DGGML_VULKAN=ON -DGGML_KOMPUTE=OFF

# Launch with manual tuning
./llama-cli \
  -m model-Q4_K_M.gguf \
  -ngl 35 \          # Layers on GPU (tune: 30-40 for 4GB)
  -cb \              # Chat-banner
  -flash-attn \      # Flash attention for memory reduction
  -t 4 \             # CPU threads (match physical cores)
  -c 2048 \          # Context size (keep under 4K for 4GB)
  --backend cuda     # Try vulkan if CUDA slower on your specific card
```

### SM75-Specific Kernel Notes

- No FP16 buffer storage on Turing — workarounds exist but add overhead
- No Tensor Cores — GEMM uses CUDA cores, not Tensor Core units
- Cooperative matrix via Vulkan (VK_KHR_cooperative_matrix) is the best path for matrix math
- `GGML_CUDA_FORCE_MMQ=1` can force matrix multiplication offload to GPU even for small tensors
- Turing has `int dot` product support — useful for INT8 quantization kernels

---

## Research Gaps & Unresolved Questions

1. **SM75 FlashAttention throughput:** Claims of "negligible speedup" on old GPUs lack rigorous benchmarking across different model sizes on SM75 specifically
2. **Vulkan vs CUDA on SM75:** The GTX 1060 benchmarks show Vulkan winning, but GTX 1650 has higher clock speeds and different architecture — need SM75-specific comparison
3. **Eigen Attention on consumer GPUs:** No benchmarks for low-rank attention on 4GB VRAM setups
4. **KV-Compress / RocketKV on consumer hardware:** All benchmarks on A100; consumer GPU deployment unknown
5. **Multi-GPU with mixed generations:** Combining GTX 1650 + older GPU for tensor parallelism untested in literature
6. **Long-context on 4GB:** LeoAM, ALISA, CacheTune assume multi-tier memory hierarchies not available on single 4GB cards

---

## Sources & Retrieval Metadata

| Source | Type | Retrieved |
|--------|------|-----------|
| arXiv:2601.14277 — GGUF quantization evaluation | Paper | 2026-09-16 |
| arXiv:2604.26334 — Pipelined Sharding (MLSys 2026) | Paper | 2026-09-16 |
| arXiv:2607.10183 — ATSInfer | Paper | 2026-09-16 |
| arXiv:2408.05646 — Eigen Attention | Paper | 2026-09-16 |
| arXiv:2410.00161 — KV-Compress | Paper | 2026-09-16 |
| arXiv:2502.14051 — RocketKV | Paper | 2026-09-16 |
| arXiv:2403.17312 — ALISA | Paper | 2026-09-16 |
| arXiv:2506.20187 — LeoAM | Paper | 2026-09-16 |
| arXiv:2605.24022 — CacheTune | Paper | 2026-09-16 |
| arXiv:2603.13289 — RelayCaching | Paper | 2026-09-16 |
| arXiv:2601.06007 — Prompt Caching Evaluation | Paper | 2026-09-16 |
| arXiv:2506.24022 — Speculative Decoding for SLMs | Paper | 2026-09-16 |
| arXiv:2604.26334 — UNISPEC | Paper | 2026-09-16 |
| community.benhalrn.com — Ollama 4GB VRAM benchmark | Blog/Community | 2026-09-16 |
| inferencerig.com — Best GGUF for 4GB VRAM | Guide | 2026-09-16 |
| insiderllm.com — 4GB VRAM guide | Guide | 2026-09-16 |
| github.com/ggml-org/llama.cpp issues #19817, #16272 | Community | 2026-09-16 |
| aisignal.dev — AirLLM analysis | Analysis | 2026-09-16 |
| gunbark.dev — SM75 optimization diary | Blog | 2026-09-16 |
| runaihome.com — AirLLM 70B on 4GB guide | Guide | 2026-09-16 |
