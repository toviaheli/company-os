# Speculative Decoding Research: GTX 1650 4GB VRAM (Turing SM75)

**Date**: 2026-09-16
**Target**: Ornith-1.5-9B on GTX 1650 Mobile 4GB, Turing SM75, CUDA
**Baseline**: CUDA IQ2_XXS @ 20.0 tok/s gen (2800MB VRAM used)
**Goal**: Find speculative decoding approaches that work within 4GB VRAM constraint

---

## 1. Speculative Decoding Overview

Speculative decoding pairs a small **draft model** that generates candidate tokens with a larger **target model** that verifies them in a single parallel forward pass. If the draft is correct, multiple tokens are produced per step. Speedup = acceptance_length / (1 + verification_cost).

### Key constraint for 4GB VRAM
Both target model + draft model + KV cache must fit in ~4GB. The draft model itself needs ~720MB GPU VRAM minimum (per skill research). With Ornith-1.5-9B at ~2.8GB (IQ2_XXS), only ~1.2GB remains for draft + KV cache — extremely tight.

---

## 2. Techniques That Work for This Setup

### 2.1 DFlash1 (audreyt/Ornith-1.5-9B-DFlash-GGUF) — BEST CANDIDATE

- **Draft model**: audreyt/Ornith-1.5-9B-DFlash-GGUF (HuggingFace)
- **Format**: GGUF, specifically built as DFlash draft for Ornith-1.5-9B
- **Also available**: onion515/ornith-9b-dflash (Q5_K_M, 914MB) — targets Ornith-1.0-9B
- **How it works**: Block-diffusion draft model that emits a whole block of tokens per forward pass, injects target hidden states into draft attention
- **VRAM**: Draft ~720MB on GPU; target + KV cache must fit in remaining ~3.3GB
- **llama.cpp support**: PR #22105 + #25110, use `--spec-type draft-dflash`
- **Key flags**:
  ```
  --spec-type draft-dflash
  --spec-draft-model <path-to-dflash-gguf>
  --spec-draft-n-max 4-7 (tune; default 15 is too aggressive for 4GB)
  --spec-draft-n-min 2
  --spec-draft-ngl auto (or manually limit draft GPU layers)
  --cache-type-k q8_0 --cache-type-v q8_0 (reduce KV cache pressure)
  ```
- **Previous failure**: "ctx_other required" config error — this is a configuration issue, NOT a fundamental incompatibility
- **What went wrong**: DFlash requires a `ctx_other` GPU context for the draft model. On 4GB VRAM, both target and draft must be explicitly managed — draft model layers need GPU offload but target KV cache may need to spill to CPU
- **Fix attempt**: Use `--spec-draft-ngl 1` or `--spec-draft-ngl auto` to limit draft GPU layers, ensuring target model + KV cache fit in remaining VRAM

**Action**: Retry DFlash1 with explicit VRAM management. Target model at ngl=99 (all on GPU), draft at ngl=1-2 (minimal GPU), KV cache at q8_0. Monitor OOM.

### 2.2 MTP (Multi-Token Prediction) — SELF-SPECULATION

- **Model**: protoLabsAI/Ornith-1.5-9B-MTP-GGUF — has MTP drafter baked into trunk
- **Advantage**: No separate draft model needed; zero extra VRAM for draft
- **Previous failure**: OOM (5.78GB drafter exceeds 4GB)
- **Root cause from research**: The MTP context on Turing doesn't handle SSM/Mamba recurrent state (n_rs_seq=0 in draft vs n_rs_seq=2 in main context for hybrid models)
- **For Ornith-1.5-9B specifically**: Ornith is a dense model (not MoE/SSM), so the SSM issue may NOT apply. The OOM was likely due to the drafter being quantized too large (5.78GB).
- **What to test**: Use the MTP GGUF with `--spec-type draft-mtp --spec-draft-n-max 2 --spec-draft-p-min 0.0`. The p-min=0.0 is critical — any value above 0.0 causes GPU shader clock spike without draft token generation on Turing.
- **Expected**: If it fits in VRAM, MTP gives 1.5-2x speedup with zero draft model overhead

### 2.3 Prompt Lookup Drafting (ngram-based) — ZERO EXTRA VRAM

- **Approach**: Uses token history to predict next tokens; no separate model needed
- **Types in llama.cpp**: `ngram-cache`, `ngram-simple`, `ngram-map-k`, `ngram-map-k4v`, `ngram-mod`
- **VRAM cost**: ~16MB (just the n-gram hash table)
- **Speedup**: 1.5-2x for repetitive text; minimal for creative/free-form generation
- **Best for**: Code generation, structured output, any text with repeated patterns
- **Flags**:
  ```
  --spec-type ngram-cache --spec-ngram-size-n 24 --spec-ngram-size-m 48
  --spec-ngram-min-hits 2
  ```
- **ngram-mod**: Uses shared pool hash, good for dense models; reduce `--spec-ngram-mod-n-min` and `--spec-ngram-mod-n-max` for dense models
- **Zero compute cost**: Pure lookup, no additional GPU work
- **Recommendation**: Always enable alongside other speculative methods as fallback

---

## 3. Techniques That DON'T Work

### 3.1 DFlash2 — NOT AVAILABLE

- DFlash2 checkpoints only exist for **Qwen3.8-27B** and **Muse-Glimmer-30B**
- No Ornith DFlash2 drafter exists
- Even if it did, Qwen3.8-27B Q4_K_M is ~16.5GB — doesn't fit 4GB VRAM
- **Verdict**: Dead end for this setup

### 3.2 FlashAttention v2/v3 — SM75 INCOMPATIBLE

- FA2/3 requires SM80+ (Ampere+)
- SM75 FlashAttention build: 2.98 tok/s at 1 layer, OOM at 50 layers
- **Verdict**: Not viable on Turing

### 3.3 ThunderKittens — SM75 INCOMPATIBLE

- Requires Ampere+ Tensor Cores (SM80+)
- TK 2.0 dropped Ampere support entirely
- **Verdict**: Not applicable

### 3.4 EAGLE-3 Draft Models — NO Ornith CHECKPOINT

- EAGLE-3 requires trained draft checkpoints per target model
- No EAGLE-3 checkpoint exists for Ornith-1.5-9B
- **Verdict**: Not available

---

## 4. Memory Optimization Techniques (Enable Speculative Decoding)

These techniques free up VRAM to make speculative decoding possible:

### 4.1 KV Cache Quantization
- **q8_0 KV cache**: ~1.5-2x reduction vs f16, minimal quality loss
- **q4_0/q4_1 KV cache**: ~3-4x reduction, slightly more quality loss
- **Flag**: `--cache-type-k q8_0 --cache-type-v q8_0`
- **Impact on 4GB**: At 4096 ctx, f16 KV = ~1.2GB; q8_0 = ~600MB; q4_0 = ~300MB
- **Recommendation**: Use q8_0 for K, q4_0 for V to save ~900MB vs f16

### 4.2 Embedding Requantization
- **Saves ~2.6GB VRAM** by keeping embeddings in lower precision
- Supported in llama.cpp via tensor override parameters
- **Risk**: May affect output quality; test carefully

### 4.3 INT4 KV Cache + Split-KV Attention
- INT4 KV cache quantization reduces bandwidth pressure by 4x
- Pairs with Split-KV attention (not yet in mainline llama.cpp)
- **Split-KV status**: Not supported in current llama.cpp build (--split-kv flag not recognized)
- **Workaround**: Use `--cache-type-k q4_0 --cache-type-v q4_0` as alternative

### 4.4 KV Cache Compression Research (Not Yet in llama.cpp)
- **KVTC** (arxiv 2511.01815): Up to 20x compression via PCA + adaptive quantization + entropy coding. Not yet integrated into llama.cpp.
- **KVQUANT** (github syedMohib44): Attention-aware 2-4 bit KV quantization. Research code, not production-ready for llama.cpp.
- **UltraQuant** (arxiv 2606.20474): 4-bit KV caching with FP4 micro-tensor approximation. AMD-focused, not yet for CUDA/Turing.
- **SpecMemo** (arxiv 2506.01986): Device-aware speculative decoding for memory-constrained devices (8GB+ VRAM). Not directly applicable to 4GB.

---

## 5. Alternative Approaches

### 5.1 Self-Speculative Decoding (SparseSpec)
- Uses the SAME model as both draft and target
- Sparse attention for drafting (PillarAttn) selects critical tokens
- Training-free, no separate draft model needed
- Research-stage (MSys 2026), not in llama.cpp yet
- **Potential**: Could work on Ornith since it reuses the same model weights

### 5.2 UNISPEC (Training-Free Speculative Decoding)
- Device-aware calibration: measures optimal draft size per hardware
- Confidence-weighted tree expansion
- Works on RTX 3090 (24GB); no 4GB-specific testing yet
- Smaller draft sizes optimal for memory-constrained GPUs
- **Key insight**: On 4GB, draft size should be SMALL (2-4 tokens), not large (15+)

### 5.3 Custom Draft Vocabularies
- Research shows 65%→74% acceptance rate improvement
- Reduces draft model size by limiting vocabulary
- Requires custom training; not available for Ornith off-the-shelf

### 5.4 Gated DeltaNet fp16
- Architecture-independent, saves ~0.88 GiB
- Research-stage, not in llama.cpp

---

## 6. Hardware-Specific Issues for Turing SM75

| Issue | Impact | Workaround |
|-------|--------|------------|
| No Tensor Cores | INT8 MMQ can corrupt output | Use INT8 for memory only, not compute |
| PCIe bandwidth 128 GB/s | CPU offloading is slow | Minimize CPU offload; keep active layers on GPU |
| 4GB VRAM limit | Very tight for target + draft + KV | Aggressive KV quantization; small draft |
| SM75 FlashAttention | FA2/3 won't compile | Use CUDA attention (--flash-attn on) |
| MTP SSM handling | n_rs_seq mismatch on hybrid models | Dense models (Ornith) may not have this issue |
| DFlash ctx_other | Draft context needs GPU allocation | Explicit layer offload control |

---

## 7. Recommended Testing Priority

### HIGH PRIORITY (test immediately)

1. **DFlash1 retry with explicit VRAM management**
   ```
   llama-server -m Ornith-1.5-9B-IQ2_XXS.gguf \
     -md Ornith-1.5-9B-DFlash-GGUF/Q5_K_M.gguf \
     --spec-type draft-dflash \
     --spec-draft-n-max 4 \
     --spec-draft-n-min 2 \
     --spec-draft-ngl 1 \
     --cache-type-k q8_0 --cache-type-v q8_0 \
     -c 4096
   ```
   - If OOM: reduce ctx to 2048, try draft at Q4_K_M
   - Monitor: `statistics spec` in logs for acceptance rate

2. **MTP GGUF with p-min=0.0**
   ```
   llama-server -m protoLabsAI/Ornith-1.5-9B-MTP-GGUF \
     --spec-type draft-mtp \
     --spec-draft-n-max 2 \
     --spec-draft-p-min 0.0 \
     -c 4096
   ```
   - The p-min=0.0 is critical for Turing (avoids shader clock spike)
   - No separate draft model = minimal VRAM overhead

3. **Ngram-cache as always-on baseline**
   ```
   --spec-type ngram-cache --spec-ngram-size-n 24 --spec-ngram-size-m 48
   ```
   - Zero VRAM cost, zero compute cost
   - Works alongside other methods

### MEDIUM PRIORITY (if high priority succeeds)

4. **Combine ngram + DFlash1**: ngram-cache as fallback when DFlash blocks rejected
5. **Aggressive KV quantization**: q4_0 KV + q4_0 V to maximize room for draft
6. **Context reduction**: Test at ctx=2048 vs ctx=4096 vs ctx=8192 for speed/VRAM tradeoff

### LOW PRIORITY (research only)

7. **Self-speculative decoding**: Research SparseSpec approach for Ornith
8. **Custom draft vocabularies**: If community builds one for Ornith
9. **UNISPEC calibration**: Test optimal draft size for 4GB SM75

---

## 8. Performance Expectations

Based on research and similar hardware:

| Scenario | Expected Speedup | Notes |
|----------|-----------------|-------|
| DFlash1 (40% acceptance, 4-token draft) | 1.3-1.6x | 20 → 26-32 tok/s |
| MTP (50% acceptance, 2-token draft) | 1.2-1.4x | 20 → 24-28 tok/s |
| Ngram-cache (repetitive text) | 1.3-1.8x | Varies by content |
| Combined ngram + DFlash1 | 1.5-2.0x | Best case with good acceptance |
| Conservative VRAM fit (q4_0 KV, small draft) | Baseline +0.5-1.0x | May be marginal |

**Reality check**: On 4GB VRAM, the VRAM constraint is the primary bottleneck. Even if acceptance rate is good, if the setup OOMs or thrashes to CPU, speed will be worse than baseline.

---

## 9. What NOT to Test (Already Documented as Failed)

- Vulkan backend (3.0 tok/s — 6-7× slower than CUDA)
- SM75 FlashAttention build (2.98 tok/s at 1 layer, OOM at 50 layers)
- MTP with separate drafter (OOM — 5.78GB exceeds 4GB)
- DFlash1 with default config ("ctx_other required" error)
- Split-KV attention (not supported in current llama.cpp)
- Continuous batching (marginal 0.1 tok/s gain)
- FlashAttention v2/v3 (SM80+ only)
- ThunderKittens (SM80+ only)

---

## 10. Key Sources

- llama.cpp speculative decoding docs: github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md
- DFlash1 for Ornith: huggingface.co/audreyt/Ornith-1.5-9B-DFlash-GGUF
- Ornith MTP GGUF: huggingface.co/protoLabsAI/Ornith-1.5-9B-MTP-GGUF
- Ornith DFlash draft: huggingface.co/onion515/ornith-9b-dflash
- DFlash2 (Qwen3.8-27B only): huggingface.co/z-lab (no Ornith drafter)
- DFlash on AMD ROCm: rocm.blogs.amd.com (block-diffusion benchmarks)
- SpecMemo paper: arxiv.org/abs/2506.01986 (memory-constrained speculative decoding)
- UNISPEC paper: aclanthology.org/2026.acl-long.285 (device-aware calibration)
- KVQUANT: github.com/syedMohib44/kvquant
- DFlash llama.cpp PR: github.com/ggml-org/llama.cpp/pull/22105
- llama.cpp DFlash2 PR: github.com/ggml-org/llama.cpp/pull/25110
- Turing SM75 MTP issue: github.com/ggml-org/llama.cpp/issues/24670
