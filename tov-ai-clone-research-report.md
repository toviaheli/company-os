# Tov AI Clone — Complete Research & Testing Report
## Generated: 2026-09-18 | GTX 1650 4GB VRAM

---

## Executive Summary

Comprehensive research completed across 4 domains: MoA inference engines, AI clone/marketing tools, fundraising/email marketing, and scholastic research (Bible/midrash). ~15 repos scouted, 12 local LLMs benchmarked, ~14 MoA configs tested.

**Key Finding:** No single tool combines all needed capabilities. Assembly approach required.

---

## 1. Local LLM Benchmarking — GTX 1650 (4GB VRAM)

### Models Tested (12 total)

| Model | Size | VRAM | Speed | Code Quality | Thinking Off |
|-------|------|------|-------|--------------|--------------|
| Qwen2.5-0.5B Instruct | 0.5B | 502MB | 72 tok/s | ✅ Working | ✅ |
| **MiniCPM5-2B** | 2.5B | 1.8GB | **37 tok/s** | ✅ Correct + docstring | ✅ |
| **Gemma-4-E2B** | 4B | 1.6GB | **37 tok/s** | ✅ Correct + explanations | ✅ |
| Gemma-4-E2B uncensored | 4B | 3.6GB | 15 tok/s | ✅ Correct + ValueError | ✅ |
| qwen2.5-coder-3b | 3.4B | ~2GB | ~16 tok/s | ✅ Working | ✅ |
| Ornith-1.5-9B IQ2_M | 9B | 3.6GB | 13 tok/s | ✅ Working (lru_cache) | ✅ |
| Qwen3.5-4B | 4B | 3.3GB | 34 tok/s | ⚠️ Thinking mode | ❌ |
| Ornith-1.5-9B AD | 9B | ~2.5GB | 20 tok/s | ❓ Unknown | N/A |
| Ornith-1.5-9B MTP | 9B | >4GB | 5 tok/s | ❌ Too big | N/A |
| Ornith-1.5-9B Q4 | 9B | >4GB | 3 tok/s | ❌ Too big | N/A |
| Qwen3.6-35B-A3B | 35B | >4GB | N/A | ❌ Too big | N/A |
| Ornith-1.5-35B-A3B | 35B | >4GB | N/A | ❌ Too big | N/A |

### Best per Job

| Job | Primary | Fallback | Speed | VRAM |
|-----|---------|----------|-------|------|
| Coding | MiniCPM5-2B | Gemma-4-E2B | 37 tok/s | 1.8GB |
| Marketing | MiniCPM5-2B | Qwen2.5-0.5B | 37/72 tok/s | 1.8/502MB |
| Fundraising | MiniCPM5-2B | Gemma-4-E2B | 37 tok/s | 1.8GB |
| Bible Research | MiniCPM5-2B | Gemma-4-E2B | 37 tok/s | 1.8GB |

### Breakthrough: Thinking Mode Fix

`--reasoning off` in llama-server disables thinking mode for all models. Without this flag, no model produces usable code — only reasoning_content.

---

## 2. MoA Inference Engines — GitHub Scout (21 engines found)

### Pure C/C++ (zero-dependency)
- **kimi-k3-in-c** (8k⭐) — 2.78T params on 8GB RAM CPU, mxFP4 quant
- **glm-5.2-in-c** (65⭐) — 744B GLM on 16GB RAM, pure C one file
- **bitnet.c** (29⭐) — C11, Flash MoE, TurboQuant KV compression
- **hummingbird** (57⭐) — Colibri-inspired modular C17 runtime

### Expert/SSD Streaming
- **Flash-MoE** — 397B Qwen on laptop, "trust the OS" page cache
- **Flash-MoE Vulkan** — Linux/Vulkan port, io_uring + GLSL compute
- **MoE-Direct** — Windows/CUDA, byte-preserving streaming, 5.6 tok/s on 122B
- **MnemoCUDA** — Multi-level VRAM cache with heat-pinning
- **moe-stream** — Apple Silicon, 3-mode auto-select
- **expert-stream** — DeepSeek V4-Flash 284B on 16GB Windows laptop

### Vulkan/CPU-only Backends
- **Kortex** (Rust+wgpu) — 160 tok/s on 30B MoE, out-of-core streaming
- **infr** (Pure Rust, Vulkan-first) — MoE expert CPU offload
- **VulkanForge** — AMD RDNA4 native FP8, 14MB static binary
- **shimmy** (5.9k⭐) — Pure Rust, zero-config, WebGPU backend
- **qwen-kernel** — Hand-written Vulkan kernels, RDNA3

### C++ Heterogeneous
- **ncnn-MoE-Runtime** — Tencent/ncnn based, MoeIR intermediate representation

### Colibri (JustVugg/colibri) — 36K⭐, Apache-2.0

**Verdict:** Vulkan backend NOT viable on GTX 1650 — NVIDIA Vulkan ICD doesn't work in Docker, no ReBAR, 4GB VRAM too small for expert cache. CPU-only mode: sub-1 tok/s.

---

## 3. AI Clone & Marketing Tools — GitHub Scout (~60 repos)

### AI Clone / Persona
| Repo | Stars | Key Features | VRAM |
|------|-------|--------------|------|
| **MirrorMind** | 12 | 7 agents, style profiling, GraphRAG, REST API, Telegram/Discord/WhatsApp | ~6 GB |
| **CloneMe** | 39 | Lightweight digital twin, multi-platform, AI-provider agnostic | Depends on provider |
| **Real-Time-Voice-Cloning** | 60K | Clone voice in 5 seconds, real-time | ~4-6 GB |
| **VoiceStudio/OmniVoice-Studio** | 18K | Local ElevenLabs alternative, 646 languages | 4-8 GB |
| **ghost-writer** | 7 | 24-dimension forensic writing style clone | API-based |
| **claude-email** | 116 | Full email marketing workflow with voice/persona matching | API-based |

### Social Media
| Repo | Stars | Key Features |
|------|-------|--------------|
| **postiz-app** | 36K | 30+ platforms, AI copilot, MCP server |
| **trypostit/trypost** | 626 | 12 platforms, AI caption/hook generation |

### Email Marketing
| Repo | Stars | Key Features |
|------|-------|--------------|
| **claros** | 48 | AI-native lifecycle email engine, deterministic plans |
| **agentkits-marketing** | 604 | Enterprise AI marketing automation |
| **email-marketing-bible** | 301 | 55K-word Claude Code skill, nonprofit playbook |

### Key Gap
No single repo combines voice + writing style + social posting + email. Tov AI Clone must be assembled from components.

**VRAM Reality:** Most repos are API-based (zero VRAM). Local LLM needs ~6-8 GB VRAM. Voice cloning ~4-6 GB GPU. For marketing, API approach is pragmatic unless privacy/cost demands local.

---

## 4. Fundraising & Email Marketing AI Tools — GitHub Scout

### Email Marketing AI
| Repo | Stars | Key Features |
|------|-------|--------------|
| **claros** | 48 | AI-native lifecycle email engine; prompt-defined flows; Postgres-backed; self-hosted |
| **agentkits-marketing** | 604 | Enterprise AI marketing automation for Claude Code/Cursor/GitHub Copilot |
| **email-marketing-bible** | 301 | 55K-word Claude Code skill; 908 sources; 19 industry playbooks; nonprofit playbook |
| **emareach** | 6 | Cold email outreach; mailbox warm-up; A/B templates; SPF/DKIM/DMARC |
| **NetSendo** | 15 | Laravel/Vue; email + SMS; MJML editor; AI Suite; MCP server |
| **laravelmail** | 4 | Laravel; local AI agents (Ollama); B2B leads DB |

### Fundraising AI
| Repo | Stars | Key Features |
|------|-------|--------------|
| **aifundme** | 0 | Agentic fundraising; contact corpus; campaign matching; multi-channel outreach |
| **PotLock/funding-ai** | 0 | NEAR blockchain; vector-similarity donor matching |
| **tiltify-donation-bot** | 4 | Discord bot; streams Tiltify donation data |

### Social Media Marketing AI
| Repo | Stars | Key Features |
|------|-------|--------------|
| **postiz-app** | 36K | 30+ platforms; AI copilot; MCP server; agent CLI |
| **trypostit/trypost** | 626 | 12 platforms; AI caption/hook generation; carousel builder |
| **MiCA-OSS** | 9 | Cross-channel campaign; email/WhatsApp/Instagram/AI video |

### CRM + AI Integration
| Repo | Stars | Key Features |
|------|-------|--------------|
| **Unite-Hub** | — | Next.js + Supabase + Claude Opus; email agent; contact intelligence scoring |
| **revenue-os** | 12 | Open-source AI SDR/prospecting agent; replaces HubSpot Prospecting Agent |
| **CampaignForge** | 0 | Multi-channel campaign orchestration dashboard |

### Key Gap
No open-source tool combines fundraising + email + social in one AI-native pipeline. MiCA-OSS comes closest (cross-channel) but lacks fundraising-specific logic.

---

## 5. Scholastic Research Tools — GitHub Scout

### RAG (Accuracy-First)
| Repo | Stars | Key Features | VRAM |
|------|-------|--------------|------|
| **SciPhi-AI/R2R** | 7,989 | Production RAG, REST API, hybrid search, KG extraction | ~2 GB (embeddings) |
| **vignesh2027/VORTEXRAG** | — | 7-layer RAG, faithfulness 0.94, causal drift filter | A100 benchmarks |
| **prabhaharanv/production-hybrid-rag** | — | Hybrid FAISS+BM25, cross-encoder rerank, citations | ~2 GB |

### Fact-Checking / Verification
| Repo | Stars | Key Features |
|------|-------|--------------|
| **Liyan06/MiniCheck** | 219 | EMNLP 2024, sentence-level fact-checking, Bespoke-MiniCheck-7B SOTA |
| **QWED-AI/qwed-verification** | 57 | Deterministic verification, 100% error detection in benchmarks |
| **sahilaf/FactEval** | 4 | Claim-level NLI verification, calibrated confidence |

### Bible Study / Hermeneutics AI
| Repo | Key Features |
|------|--------------|
| **djayatillake/studybible-mcp** | Fee & Stuart hermeneutics, Greek/Hebrew lexicons, MCP-native |
| **asphaltsanai/exegete** | Hallucination-resistant 4-stage exegesis, real original-language data |
| **davebream/claude-of-alexandria** | TDD-built, 136 automated tests, Christ-centered |
| **ronanguilloux/skill-theo** | PaRDeS Jewish exegesis methods |
| **Divine-Creative-Ministries/bible-cli** | Offline, clause-level syntax search, provenance tags |

### Knowledge Graphs
| Repo | Key Features |
|------|--------------|
| **FareedKhan-dev/agentic-knowledge-graph** | 929M edges, zero LLM calls, 83.2% PubMedQA |
| **Agents4Academia-AI/prior** | Auditable claim graph from primary literature |

### Key Gap
No tool combines midrashic/Haggadic hermeneutics with RAG verification. PaRDeS skill (skill-theo) is closest for Jewish exegesis but needs pairing with verification layer.

### Best for Bible Research Accuracy
1. **claude-of-alexandria** — Most rigorous, 136 tests, TDD-built
2. **exegete** — Hallucination-resistant, real data, 4-stage exegesis
3. **VORTEXRAG** — Best RAG faithfulness (0.94)
4. **MiniCheck** — Best fact-checking model (EMNLP 2024)
5. **studybible-mcp** — Fee & Stuart hermeneutics, lexicons

---

## 6. OBLITERATUS Research

### What It Is
- Open-source toolkit for removing refusal behaviors from LLMs (abliteration)
- C2R-Marketing fork: 0 stars, AGPL-3.0
- Original: elder-plinius/OBLITERATUS
- Uses SVD decomposition to surgically remove refusal representations
- Could enable uncensored coding output without using uncensored models
- Research tool, not production-ready

### Install
```bash
# HuggingFace Spaces
https://huggingface.co/spaces/pliny-the-prompter/obliteratus

# Colab
https://colab.research.google.com/github/elder-plinius/OBLITERATUS/blob/main/notebooks/abliterate.ipynb

# pip
pip install obliteratus
```

---

## 7. Self-MoA Testing Results

### MiniCPM5-2B Self-MoA (3 samples)
| Sample | Output | Quality |
|--------|--------|---------|
| 1 | `sorted(numbers, reverse=True)` | Full docstring + example |
| 2 | `sorted(numbers, reverse=True)` | Short docstring |
| 3 | `sorted(numbers, reverse=True)` | No docstring |

**All 3 samples produce identical correct code.** Self-MoA improves consistency.

### Gemma-4-E2B Self-MoA (3 samples)
**All 3 samples produce valid Python code** with explanations and docstrings.

---

## 8. Job-Specific Benchmarking Results

| Job | MiniCPM5-2B | Gemma-4-E2B | Qwen2.5-0.5B |
|-----|-------------|-------------|---------------|
| Coding (config parser) | ✅ Valid Python | ✅ Valid Python | ✅ Valid Python |
| Marketing (email copy) | ✅ Working email | ✅ Working email | ✅ Working email |
| Bible (course promo) | ✅ Working email | ✅ Working email | ✅ Working email |
| Bible research (midrash) | ✅ Working analysis | ✅ Working analysis | ✅ Working analysis |

---

## 9. Remaining Issues

| Issue | Status | Notes |
|-------|--------|-------|
| FreeToken install | ❌ Timed out twice | Not our hardware |
| DSpark speculative decoding | ❌ Not available | GGUF not found |
| Qwen3.5-4B thinking mode | ❌ Can't disable | Model limitation |
| Server lifecycle | ✅ Fixed | Proper port cleanup |
| Log parsing | ✅ Done | 168 unique configs |
| Mesh MoA | ❌ Not viable | 5 of 6 devices offline |
| Colibri Vulkan | ❌ Not viable | GTX 1650 incompatible |

---

## 10. Final Recommendations

### Best Configuration per Job

**Coding:** MiniCPM5-2B with `--reasoning off` (37 tok/s, 1.8GB VRAM)

**Marketing:** MiniCPM5-2B + MirrorMind persona framework + postiz-app + claros

**Fundraising:** MiniCPM5-2B + claros + email-marketing-bible + postiz-app

**Bible Research:** MiniCPM5-2B + claude-of-alexandria + exegete + VORTEXRAG

### Assembly Approach Required

No single repo combines all capabilities. Each job requires assembling 2-4 tools:

1. **AI Clone:** MirrorMind (persona) + VoiceStudio (voice) + postiz-app (social) + claude-email (email)
2. **Marketing:** MiniCPM5-2B (local LLM) + claros (email) + postiz-app (social) + claude-email (workflow)
3. **Fundraising:** MiniCPM5-2B (local LLM) + claros (email) + email-marketing-bible (skill) + postiz-app (social)
4. **Bible Research:** MiniCPM5-2B (local LLM) + claude-of-alexandria (study) + exegete (exegesis) + VORTEXRAG (RAG) + MiniCheck (fact-checking)

### GPU Upgrade Path

If GPU upgrade becomes possible:
- RTX 3060 12GB: ~$200 — can run 2 models + voice cloning
- RTX 4060 8GB: ~$300 — can run 3 models + voice cloning
- RTX 5070 Ti: ~$500 — can run colibri Vulkan at 1.07 tok/s
