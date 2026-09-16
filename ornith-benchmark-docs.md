# Ornith-1.5-9B LLM Server Testing — Complete Documentation

## Hardware
- Acer Nitro AN515-54, CachyOS 7.2.2-1
- GTX 1650 Mobile 4GB VRAM, 23GB RAM
- Kernel 7.2.2, Limine bootloader

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

## Real-World Research (2026-09-16)

### Key Finding: No 9B Model Fits in 4GB VRAM at Usable Quality

Per WillItRunAI and web research:
- Q4_K_M quants need 5.1-5.7 GB VRAM for 9B models
- Only IQ2_XXS/IQ2_S/IQ1_M quantization fits in 4GB, but quality is severely degraded
- AD-IQ2_S-IQ2_XS: 71% top-1 token agreement (degraded)
- AD-IQ2_XXS-IQ1_M: even more degraded

### Best Models for 4GB VRAM (GTX 1650)
| Model | VRAM | Tok/s | Tier |
|-------|------|-------|------|
| Ministral 3 3B | 3.9 GB | 38 | A |
| Qwen 3.5 2B | 4.2 GB | 28 | B |
| Qwen 3 1.7B | 4.0 GB | 24 | B |
| Qwen 2.5 Coder 1.5B | 2.6 GB | 21 | B |
| DeepSeek R1 1.5B | 2.6 GB | 21 | B |
| Ornith IQ2_S (3.3GB) | 3.3 GB | 17.5 | C |
| Ornith IQ2_XXS (2.8GB) | 2.8 GB | 8.7 | C |

### Ornith 1.0-9B on 16GB GPU (RTX 5070 Ti)
- Q4_K_M: ~200 tok/s, 400K context with hybrid attention + Q8 KV cache
- 5.5 GB weights + 6.4 GB KV cache + 1 GB overhead = 14.5 GB total

### llama-server vs Ollama
- Raw throughput within 3-12% for single-user workloads
- llama-server wins by 10-20% when hand-tuned with --flash-attn, KV-cache quantization, explicit GPU offload
- Ollama resurfaces prefill cache automatically; llama-server requires manual config

## Scripts Created
- /home/tov/models/bench_ornith_repro.py — Reproduce Ornith benchmark
- /home/tov/models/bench_ornith_cpu.py — CPU benchmark script
- /home/tov/models/benchmark_all.py — Full benchmark suite

## Ollama Models (also tested)
- ornith9b-iq2xxs on Ollama: 13.2 tok/s
- nomic-embed-text: embedding model, wrong API for chat

## Conclusion

**Best Ornith result: 17.5 tok/s** with ngl=99 (auto-detect) on llama-server. The model fits in 4GB VRAM with auto-detection. Manual ngl settings are suboptimal.

**4GB VRAM is the hard limit.** No 9B model runs at usable quality on 4GB VRAM. The best path forward is either:
1. Accept Ornith IQ2_S at 17.5 tok/s (degraded quality)
2. Switch to a smaller model (Ministral 3 3B at 38 tok/s)
3. Upgrade to 8GB+ VRAM GPU for Q4_K_M quants