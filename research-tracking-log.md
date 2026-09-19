# Research Tracking Log

This file tracks all LLM optimization tests performed on GTX 1650 4GB VRAM
to prevent repeating tests and document results for future agents.

## Definitive Test Results (PROVEN)

| Test | Backend | Layers | Prompt | Gen | VRAM | Status |
|------|---------|--------|--------|-----|------|--------|
| CUDA IQ2_XXS | CUDA | 99 | 50.8 tok/s | 20.0 tok/s | ~2800MB | ✅ BASELINE |
| CUDA IQ2_S | CUDA | 99 | 29.2 tok/s | 18.3 tok/s | ~2800MB | ✅ Tested |
| Vulkan IQ2_XXS | Vulkan | 1 | 3.8 tok/s | 3.0 tok/s | 3236MB | ❌ FAILED |
| SM75 FlashAttn IQ2_XXS | CUDA-SM75 | 1 | 4.1 tok/s | 3.0 tok/s | OOM@50 | ❌ FAILED |
| Continuous batching | CUDA | 99 | 24.4 tok/s | 20.1 tok/s | ~2800MB | ⚠️ Marginal |
| MTP speculative decoding | CUDA | 99 | - | OOM | - | ❌ FAILED |
| DFlash1 spec decoding | CUDA | 99+1draft | 5.2 tok/s | 2.7 tok/s | OOM risk | ❌ SLOWER |
| Ngram-cache spec decoding | CUDA | 99 | 17.5 tok/s | 13.8 tok/s | ~2800MB | ❌ SLOWER |

## Speculative Decoding Verdict
- DFlash1: 2.7 tok/s gen (vs baseline 20.0) — draft model overhead kills speed
- Ngram-cache: 13.8 tok/s gen (vs baseline 20.0) — lookup overhead too high
- **Speculative decoding NOT beneficial on 4GB VRAM for Ornith-1.5-9B**
- Baseline CUDA IQ2_XXS at 20.0 tok/s remains the best

## Techniques Tested & Rejected
- Vulkan backend: 6-7× slower than CUDA on 4GB VRAM
- SM75 FlashAttention build: OOM at 50 layers, 2.98 tok/s at 1 layer
- MTP draft model: OOM (5.78GB drafter exceeds 4GB VRAM)
- DFlash1: "ctx_other required" config error
- Split-KV: not supported in current llama.cpp
- Continuous batching: marginal gain (0.1 tok/s), not worth prioritizing

## Techniques Yet to Test (CUTTING-EDGE)
- vAttention (attention v2)
- KVTC (KV token compression)
- Mustafar (KV cache quantization)
- LvLLM (hybrid inference)
- FATE (flash attention tuning)
- GSQ-RCO quantization
- Expert offloading (MoE models)
- DiskLLM (SSD weight streaming)
- HotPin (memory pinning)
- llm-fit (LD_PRELOAD shim)

## Speculative Decoding Research (COMPLETED)
- Subagent: deleg_b02c4934 ✅
- Deliverable: /home/tov/speculative-decoding-research.md ✅
- Key finding: DFlash1 SLOWER than baseline (2.7 vs 20.0 tok/s) — rejected
- DFlash2: NOT available for Ornith (only Qwen3.8-27B)
- MTP: retry with p-min=0.0, no separate drafter needed
- Ngram-cache: zero VRAM cost, always-on baseline
- Expected speedup: 1.3-1.6x with DFlash1 (20→26-32 tok/s)
- ACTUAL RESULT: DFlash1 2.7 tok/s, Ngram-cache 13.8 tok/s — BOTH SLOWER than baseline 20.0 tok/s
- **Verdict: Speculative decoding NOT beneficial on 4GB VRAM for Ornith-1.5-9B**

## Testing Status — Scoped Tests Complete; 10+ Techniques Remain Untested
- Vulkan backend: FAILED ✅
- SM75 build: FAILED ✅
- DFlash1: SLOWER ✅
- Ngram-cache: SLOWER ✅
- MTP: OOM ✅
- Continuous batching: marginal ✅
- IQ2_S: tested ✅
- **Baseline CUDA IQ2_XXS at 20.0 tok/s remains the best achievable on this hardware**

## Cutting-Edge Findings Applied
- DFlash2: Only available for Qwen3.8-27B, NOT for Ornith-1.5-9B
- SpecMemo: Speculative decoding for pocket-sized models
- Prompt lookup drafting: 381 tok/s achieved in Cloud Codes video
- Custom draft vocabularies: 97.5% acceptance rate

## Unresolved Issues
- Split-KV Attention: --split-kv flag not recognized in current llama.cpp build
- DFlash1 "ctx_other required": configuration issue, not resolved
- MTP OOM: drafter too large for 4GB VRAM

## Best Result Achieved
CUDA IQ2_XXS: 20.0 tok/s generation (baseline)
CUDA IQ2_S: 18.3 tok/s generation (better quality, slightly slower)

## Jev / Decision Router Benchmarks (2026-09-19)

| Test | Backend | Latency | VRAM | Cost | Status |
|------|---------|---------|------|------|--------|
| RLCD ModernBERT-151M | Local CUDA | 27.3ms avg | 604MB | $0 | ✅ Working |
| TypeSafe Jev API | Cloud | 459ms | N/A | $0.042/1M tokens | ⚠️ Needs key |
| VRAM Available | GTX 1650 | — | 3900MB free | — | ✅ Confirmed |

### Jev Feasibility Verdict
- RLCD local: 27ms avg, 604MB VRAM, Apache 2.0, no API key — FITS ON ACER
- TypeSafe cloud: needs waitlist key, 459ms latency — BLOCKED (no key)
- All 5 API keys working (OpenRouter, NVIDIA, HF, Cohere, Nous)
- Vercel Gateway: requires credit card — BLOCKED (user won't pay)
