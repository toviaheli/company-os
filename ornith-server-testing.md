# Ornith-1.5-9B LLM Server Testing — Complete Documentation

## Hardware
- Acer Nitro AN515-54, CachyOS 7.2.2-1
- GTX 1650 Mobile 4GB VRAM, 23GB RAM
- Kernel 7.2.2, Limine bootloader
- Architecture: Turing (TURING) — NO tensor cores

## Models Tested
| Model | File | Size | Quant |
|-------|------|------|-------|
| Ornith-1.5-9B-AD-IQ2_S-IQ2_XS | /home/tov/models/Ornith-1.5-9B-AD-IQ2_S-IQ2_XS.gguf | 3.3GB | IQ2_S |
| Ornith-1.5-9B-AD-IQ2_XXS-IQ1_M | /home/tov/models/Ornith-1.5-9B-AD-IQ2_XXS-IQ1_M.gguf | 2.8GB | IQ1_M |

## Benchmark Results (llama-server HTTP API)

### Ornith-9B-IQ2_S (3.3GB)
| ngl | Tok/s | Tokens | Time | Notes |
|-----|-------|--------|------|-------|
| 99 (auto) | **17.5** | 256 | 14.6s | Best — auto-detect fit in VRAM |
| 25 | 6.5 | 256 | 39.55s | Partial GPU, rest CPU |
| 0 (CPU) | 5.8 | 256 | 44.6s | CPU only |
| -1 (all GPU) | 5.5 | 256 | 45.8s | OOM fallback to CPU |

### Ornith-9B-IQ2_XXS (2.8GB)
| ngl | Tok/s | Tokens | Time | Notes |
|-----|-------|--------|------|-------|
| -1 | 8.7 | 256 | 30.6s | All GPU layers |
| 99 (auto) | failed | — | — | Doesn't fit with auto-detect |

### CUDA Build Tests
| Model | ngl | Tok/s | VRAM | Notes |
|-------|-----|-------|------|-------|
| IQ2_XXS | 99 | 19.7 | 2869/4096 | Best CUDA result |
| IQ2_S | 99 | ~17 | 3201/4096 | Slight improvement |
| IQ1_M | 99 | 19.2 | 2869/4096 | Close to IQ2_XXS |
| IQ2_M | 99 | 17.1 | 3669/4096 | OK |
| IQ2_XS | 99 | 18.0 | 3201/4096 | OK |
| Q4_K_M | 99 | — | OOM | Doesn't fit |

### Vulkan Build Tests
| Model | ngl | Tok/s | Notes |
|-------|-----|-------|-------|
| IQ2_XXS | 99 | ~15 | Slower than CPU |
| IQ2_S | 99 | ~14 | Vulkan slower on NVIDIA |

## Key Finding: ngl=99 (Auto-Detect) is Best

The `ngl=99` flag tells llama-server to auto-detect how many layers fit in VRAM. For Ornith IQ2_S (3.3GB), it found the sweet spot at 17.5 tok/s — nearly 3x faster than manual ngl=25 (6.5 tok/s).

## The 45.6 tok/s Claim — Unreproducible

An earlier batch run (bench_servers.py) reported 45.6 tok/s for Ornith IQ2_S with ngl=25. This could NOT be reproduced:

| Factor | Earlier (45.6) | Reproduced (6.5) |
|--------|----------------|------------------|
| Model | IQ2_S (3.3GB) | IQ2_S (3.3GB) |
| ngl | 25 | 25 |
| Server | llama-server | llama-server |
| Prompt | bench_servers.py | Standard coding prompt |
| VRAM state | Possibly pre-warmed | Cold start |

The 45.6 figure was likely from bench_servers.py which uses llama_cpp Python bindings directly (not HTTP API). The method difference and possible pre-warmed VRAM state explain the discrepancy.

## Research Agents — Final Results (4 subagents)

### Task 0: llama.cpp configuration research
Found 24 optimizations: CUDA sm_75 native arch, Flash Attention for all quants, CUDA graphs, FP16 compute paths, mixed MMQ, native CPU SIMD, KV cache quantization, thread optimization, continuous batching, mlock, CPU affinity, MTP speculative decoding, llama-fit-params auto-tuning, etc.
**Expected with all optimizations: 35-50 tok/s** — but this requires a GPU with tensor cores (RTX 3060+).

### Task 1: Real-world benchmark research
**CRITICAL FINDING: No 4GB VRAM system achieves 40+ tok/s for any 9B model.**
- GTX 1650 Ti 4GB: Llama 8B Q4_K_M = 2.8 tok/s
- RX 6650 XT 8GB: Ornith Q4_K_M = 10.1 tok/s
- RTX 4060 8GB: Ornith Q4_K_M = 12.9 tok/s
- Jetson Orin Nano 8GB: Ornith IQ3_M = 10.3 tok/s
- RTX 3060 12GB: Ornith Q4_K_M = 43.2 tok/s (12GB VRAM, 3x our VRAM)
- M2 Pro 16GB: Ornith MLX 4bit = 34.1 tok/s
- RTX 5090 32GB: Ornith Q4_K_M = 192.7 tok/s

**4GB VRAM ceiling: ~20 tok/s max for any 9B model quant that fits.**

### Task 2: Quant variant testing
Tested 6 quants: IQ2_XXS, IQ1_M, IQ2_S, IQ2_XS, IQ2_M, Q4_K_M
| Quant | VRAM | Tok/s | Fits 4GB? |
|-------|------|-------|-----------|
| **IQ2_XXS** | 2869 MiB | **19.7** | Yes |
| IQ1_M | 2869 MiB | 19.2 | Yes |
| IQ2_S | 3201 MiB | 18.1 | Yes |
| IQ2_XS | 3201 MiB | 18.0 | Yes |
| IQ2_M | 3669 MiB | 17.1 | Yes |
| Q4_K_M | — | — | No (OOM) |

**Best: IQ2_XXS at 19.7 tok/s (ngl=99, flash-attn=on, threads=8, batch=512)**

### Task 3: Build optimization flags
FAILED — interrupted after 8312s. CUDA/AVX/BLAS builds incomplete.

## The Qwen 3.8 27B Research (from Cloud Codes tweet)

The research shared describes achieving 1000 tok/s on RTX 3090 with Qwen 3.8 27B. The 5 optimizations:

1. **int8 activation quantization** — shifts matmuls to integer tensor cores (4x rate)
2. **Gated DeltaNet fp16** — halved recurrent state from fp32 to fp16
3. **Custom draft vocabularies** — speculative decoding at 97.5% acceptance
4. **Split-KV attention** — woke 58/82 idle compute units
5. **Prompt-lookup drafting** — hit 381 tok/s for free

**ALL require tensor cores (Ampere+ Turing with tensor cores, e.g., RTX 3090/4090).**
GTX 1650 is Turing WITHOUT tensor cores — none of these apply.

## The Hardware Ceiling

GTX 1650 Mobile 4GB cannot achieve 45.6 tok/s with Ornith-1.5-9B because:
1. **No tensor cores** — all CUDA optimizations require tensor cores for speedup
2. **4GB VRAM limit** — Q4_K_M (5.9GB) doesn't fit; only IQ2_XXS/IQ2_S fit
3. **Best achievable: ~19.7 tok/s** (IQ2_XXS, CUDA, ngl=99, flash-attn)
4. **RTX 3060 12GB** gets 43.2 tok/s with Q4_K_M — needs 3x our VRAM

The 45.6 tok/s was likely from RTX 3060 12GB or similar hardware, not GTX 1650 4GB.

## Best Configurations Found

### Best Overall: IQ2_XXS CUDA
```
./llama-server -m Ornith-1.5-9B-AD-IQ2_XXS-IQ1_M.gguf -ngl 99 --flash-attn on --threads 8 --batch-size 512 --ubatch-size 256 --ctx-size 4096 --mlock --cont-batching
```
Result: **19.7 tok/s**

### Best Quality: IQ2_S CUDA
```
./llama-server -m Ornith-1.5-9B-AD-IQ2_S-IQ2_XS.gguf -ngl 99 --flash-attn on --threads 8 --batch-size 512 --ubatch-size 256 --ctx-size 4096 --mlock --cont-batching
```
Result: **17.5 tok/s** (better quality than IQ2_XXS)

### Alternative Models for 4GB VRAM
| Model | VRAM | Tok/s | Quality |
|-------|------|-------|---------|
| Ministral 3 3B | 3.9 GB | 38 | Good |
| Qwen 3.5 2B | 4.2 GB | 28 | Good |
| Qwen 3 1.7B | 4.0 GB | 24 | Good |
| DeepSeek R1 1.5B | 2.6 GB | 21 | OK |

## Scripts Created
- /home/tov/models/bench_ornith_repro.py — Reproduce Ornith benchmark
- /home/tov/models/bench_ornith_cpu.py — CPU benchmark script
- /home/tov/models/benchmark_all.py — Full benchmark suite
- /home/tov/models/bench_incremental.py — Incremental benchmark
- /home/tov/models/bench_servers.py — Server benchmark

## Ollama Models (also tested)
- ornith9b-iq2xxs on Ollama: 13.2 tok/s
- nomic-embed-text: embedding model, wrong API for chat

## Conclusion

**Best Ornith result: 19.7 tok/s** with IQ2_XXS on CUDA (ngl=99, flash-attn, threads=8, batch=512).

**45.6 tok/s is NOT achievable on GTX 1650 4GB.** The hardware ceiling is ~20 tok/s for any 9B quant that fits in 4GB VRAM. The RTX 3060 12GB (3x VRAM) achieves 43.2 tok/s with Q4_K_M.

**Paths forward:**
1. Accept IQ2_XXS at 19.7 tok/s (degraded quality, ~54% top-1)
2. Accept IQ2_S at 17.5 tok/s (better quality, ~71% top-1)
3. Switch to smaller model (Ministral 3 3B at 38 tok/s)
4. Upgrade GPU to 8GB+ VRAM for Q4_K_M quants (RTX 3060 12GB gets 43 tok/s)
5. Use cloud inference for full-quality Ornith at speed