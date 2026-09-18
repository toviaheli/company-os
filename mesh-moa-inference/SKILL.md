---
name: "mesh-moa-inference"
description: "Use when setting up mesh MoA inference."
---

# Mesh MoA Inference Skill

## Tailscale Mesh Setup

### Architecture
- Tailscale creates WireGuard mesh VPN between devices
- Each device gets stable 100.x.x.x address
- No port forwarding, no cloud accounts, no VPN config beyond install
- Ollama on GPU box serves models, other devices connect via Tailscale IP
- Direct peer-to-peer, encrypted, NAT traversal automatic

### Setup Steps
1. Install Tailscale on all devices: `curl -fsSL https://tailscale.com/install.sh | sh`
2. Authenticate: `sudo tailscale up`
3. Install Ollama on GPU box: `curl -fsSL https://ollama.com/install.sh | sh`
4. Configure Ollama to listen on all interfaces
5. Allow Tailscale traffic: `sudo ufw allow in on tailscale0`
6. From client devices: `curl http://<tailscale-ip>:11434/v1/models`
7. Point Qwen Code CLI at shared box via OpenAI-compatible endpoint

### Devices Available (Tailscale)
| Device | IP | OS | Status |
|--------|-----|-----|--------|
| cachyos-x8664 | 100.108.98.69 | linux (GTX 1650) | online |
| michelles-a71 | 100.100.160.87 | android | online |
| tov-oem-aspiree5575 | 100.73.108.50 | linux | online |
| tov-oracle-cloud | 100.124.140.111 | linux | online |
| 404-not-found-wsl | 100.118.117.104 | linux | offline |
| desktop-firewing | 100.109.186.11 | windows | offline |
| fwin232-plus | 100.119.177.50 | windows | offline |
| moto-g-stylus | 100.91.86.34 | android | offline |
| system-error-404 | 100.118.244.21 | android | offline |
| tov-1 | 100.79.187.25 | linux | offline |
| tov-nitroan51554 | 100.73.44.64 | linux | offline |

## FreeToken - Edge-Native MoE Serving

### What It Does
- Edge-native MoE serving engine
- Runs 7B-53B GLM-5.2 on single workstation GPU
- 35B model at 39.3 tok/s on 8GB laptop GPU
- 753B GLM-5.2 on single workstation GPU at 14.9 tok/s
- Qwen3.6-35B-A3B at 77-83 tok/s on RTX 5090
- Apache-2.0, on PyPI, Windows/Linux desktop app

### Key Innovation
- Bandwidth-adaptive CPU-GPU co-execution
- Elastic memory management: dynamic VRAM reallocation
- Semantic-aware caching for agentic context
- Supports 20+ MoE models
- OpenAI and Anthropic compatible APIs

### Install
```bash
pip install freetoken
# Or download desktop app from flashml.ai
```

### Benchmarks
| Model | Hardware | tok/s | vs llama.cpp |
|-------|----------|-------|-------------|
| Qwen3.6-35B-A3B | RTX 5090 | 77-83 | 1.5-2.3x |
| DeepSeek-V4-Flash | RTX 5090 | 22-25 | 1.5-2.3x |
| GLM-5.2 (753B) | RTX PRO 6000 | 14.9 | 2.0x |
| 35B model | 8GB RTX 4060 | 39.3 | 1.2x |

## Hermes MoA vs Claude Fable 5

### GoldieBench Results (47 tasks)
| System | Avg Score | Wins | Losses | Ties |
|--------|-----------|------|--------|------|
| Hermes MoA | 8.17/10 | 26 | 16 | 5 |
| Claude Fable 5 | 8.10/10 | 16 | 26 | 5 |

### Hermes MoA Panel
- Default: Opus 4.8 + GPT-5.5, aggregated by Opus 4.8
- All via OpenRouter key
- Configurable panel from Mixture tab in Agent OS
- Provider-agnostic: swap any OpenRouter model

### Trade-offs
- Latency: ~110-140s per single-file build vs solo model's one call
- Cost: every panel slot + aggregator are separate calls
- 47 of 47 tasks scored (not 3 of 42)

## MoA Architectures

### Hermes MoA
- Panel of models + aggregator
- Provider-agnostic
- Beats solo Opus 4.8 by 6 points on AlpacaEval

### Self-MoA
- Single model, multiple samples
- Quality consolidation over diversity
- 6.6% improvement over standard MoA on AlpacaEval

### RouteMoA
- Dynamic routing with lightweight scorer
- 89.8% cost reduction, 63.6% latency reduction
- Mixture of judges for posterior correction

### MoMoA (Google)
- Mixture of Mixture of Agents
- Swarm of squabbling specialists
- Orchestrator + Work Phase Rooms + Experts + Overseer
- Exceeds ReAct-loop agents using Gemini Flash

## Coding Agent Benchmarks

### Best Local Coding Models (GTX 1650, --reasoning off)
| Model | VRAM | Speed | Code Quality | Coding Focus |
|-------|------|-------|--------------|--------------|
| MiniCPM5-2B Q4 | 1.8GB | 37 tok/s | ✅ Correct, docstring | ✅ SOTA 2B |
| Gemma-4-E2B Q4 | 1.6GB | 37 tok/s | ✅ Correct + explanations | ❌ General |
| Gemma-4-E2B uncensored | 3.6GB | 33 tok/s | ✅ Correct + ValueError | ❌ General |
| Qwen2.5-0.5B Instruct | 502MB | **72 tok/s** | ✅ Working code | ❌ General |
| qwen2.5-coder-3b | ~2GB | ~16 tok/s | ✅ Working code | ✅ Coder |
| Ornith-1.5-9B IQ2_M | 3.6GB | 13 tok/s | ✅ Working code | ❌ General |
| Qwen3.5-4B | 3.3GB | 34 tok/s | ⚠️ Thinking mode | ❌ General |
| Ornith-1.5-9B AD | ~2.5GB | 20 tok/s | ❓ Unknown | MoE |

### Frontier API Models
| Model | SWE-bench | VRAM | Notes |
|-------|-----------|------|-------|
| DeepSeek V4-Pro | ~80% | 80GB+ | API only, MIT license |
| GLM-5.x | ~78% | 80GB+ | API only, MIT license |
| Kimi K2.x | ~80% | 80GB+ | API only, MIT license |

## Hybrid Pattern (80/20 Rule)
- 80% routine coding locally (completions, single-file edits, tests)
- 20% cloud for complex tasks (multi-file refactors, debugging)
- GPU pays for itself in 8 months vs API costs

## Adversarial Findings

### Hard Constraints
1. GTX 1650 has 4GB VRAM — only one model at a time
2. FreeToken requires NVIDIA RTX 30/40/50 series
3. Tailscale mesh requires all devices online
4. MoA requires cloud API keys for Hermes MoA panel
5. Local MoA needs sequential server loading
6. Thinking mode produces reasoning output, not usable code
7. SSH access not available on online devices
8. **Mesh MoA assumes hardware that doesn't exist** — 5 of 6 Tailscale devices are offline ← NEW

### Critical Gaps
1. FreeToken not tested on GTX 1650 (pip install timed out)
2. Tailscale mesh not tested with LLM serving (5 of 6 devices offline)
3. Job-specific benchmarking not done
4. DSpark speculative decoding not tested (GGUF not available)
5. Self-MoA quality consolidation not tested
6. **Mesh MoA assumes hardware that doesn't exist** — 5 of 6 Tailscale devices are offline

### Open Questions
1. Does FreeToken work on GTX 1650 (4GB)?
2. Can Tailscale mesh serve Ollama models reliably?
3. What is the cost of Hermes MoA panel per task?
4. Should we use Self-MoA instead of multi-model MoA?
5. Is 20 tok/s sufficient for interactive coding?
6. Cloud vs mesh vs local: which is cheapest?
7. **What's the fastest path from 4GB GPU to useful code output?** ← RESOLVED: MiniCPM5-2B with --reasoning off

### Creative Perspective Findings
- Mesh MoA is a solution looking for a problem on 4GB VRAM
- Self-MoA (single model, multiple samples) is the practical path
- MiniCPM5-2B on llama.cpp with --reasoning off is the fastest path to working code
- Doing nothing / skipping mesh entirely might deliver more value faster
- The skill's framing treats mesh + MoA as the goal, but deployed hardware can't support it
- **Mesh MoA assumes hardware that doesn't exist** — 5 of 6 Tailscale devices are offline

## Colibri — Pure C MoE Inference Engine

### What It Is
- Pure C inference engine, zero deps, experts streamed from disk
- 36K stars, Apache-2.0 license
- Runs frontier MoE models on consumer hardware
- Supports GLM-5.2 744B, Kimi K3, DeepSeek V4 Flash, Inkling

### Key Innovation
- Streams only active experts from disk (JIT for weights)
- Per-layer LRU cache, learned pinned hot-store, optional VRAM tier
- Learns workload: hot experts get cached, cold ones stay on disk
- Single C file engine (c/colibri.c) plus small headers

### GPU Backends
- CUDA backend, VRAM expert tier, full residency
- Vulkan backend (any GPU: AMD via RADV, incl. cards ROCm dropped)
- Apple Silicon Metal backend
- CPU streaming (no GPU required)


| Hardware | tok/s | Pipeline |
|----------|-------|----------|
| RTX 5070 Ti laptop | 1.07 | GPU-resident |
| 25 GB dev box | 0.05-0.1 | Cold CPU |
| GTX 1650 (est.) | ? | Vulkan/CPU |

### Relevance to GTX 1650
- Vulkan backend could work on GTX 1650 (Vulkan 1.1 support)
- Expert streaming means not all experts need VRAM at once
- Pure C, zero deps — no Python runtime overhead
- Lower VRAM requirement than llama.cpp for MoE models
- **Potentially the best fit for 4GB VRAM** — expert streaming avoids VRAM overflow

### Install
```bash
git clone https://github.com/JustVugg/colibri.git
cd colibri
make glm  # or make kimi_k3, make deepseek-v4
```

### Pitfalls
- 1.6TB model snapshot for Kimi K3
- Requires model checkpoint conversion
- Vulkan backend untested on GTX 1650
- Single-person project, community-driven
- No formal benchmark on GTX 1650 or older GPUs

### Vulkan Backend for GTX 1650: ❌ NOT VIABLE
- NVIDIA's Vulkan ICD does NOT initialize inside Docker (standard deployment)
- GTX 1650 has no Resizable BAR — Vulkan HOST_VISIBLE allocation limited to ~256 MB
- 4GB VRAM too small for any meaningful expert cache tier
- CPU-only mode: sub-1 tok/s for GLM-5.2
- Bottom line: Vulkan not viable on GTX 1650. CPU mode only.

## GitHub Scout — 21 MoE Inference Engines Found

### Pure C/C++ (zero-dependency)
- **kimi-k3-in-c** (8k stars) — 2.78T params on 8GB RAM CPU, mxFP4 quant
- **glm-5.2-in-c** (65 stars) — 744B GLM on 16GB RAM, pure C one file
- **bitnet.c** (29 stars) — C11, Flash MoE, TurboQuant KV compression
- **hummingbird** (57 stars) — Colibri-inspired modular C17 runtime

### Expert/SSD Streaming
- **Flash-MoE** — 397B Qwen on laptop, "trust the OS" page cache
- **Flash-MoE Vulkan** — Linux/Vulkan port, io_uring + GLSL compute
- **MoE-Direct** — Windows/CUDA, byte-preserving streaming, 5.6 tok/s on 122B
- **MnemoCUDA** — Multi-level VRAM cache with heat-pinning
- **moe-stream** — Apple Silicon, 3-mode auto-select
- **expert-stream** — DeepSeek V4-Flash 284B on 16GB Windows laptop

### Vulkan/CPU-only Backends
- **Kortex** (Rust+wgpu) — 160 tok/s on 30B MoE, out-of-core streaming
- **infr** (Pure Rust, Vulkan-first) — MoE expert CPU offload via INFR_NCMOE=N
- **VulkanForge** — AMD RDNA4 native FP8, 14MB static binary
- **shimmy** (5.9k stars) — Pure Rust, zero-config, WebGPU backend
- **qwen-kernel** — Hand-written Vulkan kernels, RDNA3

### C++ Heterogeneous Runtimes
- **ncnn-MoE-Runtime** — Tencent/ncnn based, MoeIR intermediate representation

### Best Bets for GTX 1650
1. **bitnet.c** — C11, Flash MoE, TurboQuant KV compression, no GPU required
2. **hummingbird** — Colibri-inspired modular runtime, adapter-based
3. **Kortex** — Rust+wgpu, 160 tok/s on 30B MoE, out-of-core streaming
4. **infr** — Pure Rust, Vulkan-first, MoE expert CPU offload

### What's Verified vs Uncertain
- ✅ Repos confirmed live, descriptions and key features verified
- ⚠️ Star counts for several new repos (< 6 months old) may not reflect maturity
- ⚠️ Kortex and WARP GitHub API returned errors — star counts unconfirmed
- ⚠️ Most engines are pre-1.0 with limited model family support

## OBLITERATUS — Abliteration Toolkit

### What It Is
- Open-source toolkit for removing refusal behaviors from LLMs
- Implements abliteration — surgically removes refusal representations without retraining
- Fork: C2R-Marketing/OBLITERATUS (0 stars, AGPL-3.0)
- Original: elder-plinius/OBLITERATUS

### Key Features
- Map the chains — ablation studies to find refusal anchors
- Break the chains — SVD decomposition, targeted obliteration
- The informed method — analysis-guided obliteration
- Gradio HF Spaces interface, Colab notebook
- Python API for integration with evaluation harnesses

### Relevance to Our Work
- Could enable uncensored coding output without using uncensored models
- Removes safety guardrails that may block code generation
- Research tool, not production-ready
- Requires technical understanding to use responsibly

### Install
```bash
# HuggingFace Spaces (no setup)
https://huggingface.co/spaces/pliny-the-prompter/obliteratus

# Colab
https://colab.research.google.com/github/elder-plinius/OBLITERATUS/blob/main/notebooks/abliterate.ipynb

# pip
pip install obliteratus
```

## AI Clone & Marketing Tools

### AI Clone / Persona
- **WeClone-Skills** (xming521) — AI twin skill set, persona pack workflow
- **my-digital-twin** (ammonhaggerty) — AI proxy agent, digital twin
- **CloneMemBench** (AvatarMemory) — Benchmarking long-term memory for AI clones
- **Tavus** — Digital twin video generation, personalized outreach
- **D-ID** — Photo-to-video, lip-sync, voice cloning

### Marketing Automation
- **founder-fundraising-outreach** (evalyze-ai) — Claude skill for investor outreach
- **Fundraisly** — AI fundraising agent, 1479 Product Hunt votes
- **headroom** (chopratejas) — LLM cost reduction, 1265 stars
- **affaan-m/ECC** — Email campaign tool, 1533 stars

## Scholastic Research Tools (Bible)

### Rigorous Biblical Study
- **claude-of-alexandria** (davebream) — Claude plugin, TDD-built, 136 automated tests
  - 53 RED-phase tests documenting bare-model failures
  - 6 skills + 6 sub-agents, all 66 canonical books
  - Historical-grammatical method, Christ-centered
  - Psychologicalizing/moralistic drift prevention

- **studybible-mcp** (djayatillake) — Bible study MCP server
  - Greek/Hebrew lexicons (LSJ, BDB, Abbott-Smith)
  - Fee & Stuart hermeneutics method
  - Aquifer Open Study Notes, Tyndale Bible Dictionary
  - Genre-specific interpretation principles

- **ScriptureDeepDive** (CraigBuckmaster) — 70+ scholars, offline, free
  - 72 scholar commentaries, word studies, debate topics
  - DVCR quality scoring (Density/Verse Coverage/Completeness/Relevance)
  - Tier 0-3 accuracy auditor
  - CI pipeline with schema validation

- **exegete** (asphaltsanai) — Hallucination-resistant Bible exegesis
  - 4-stage exegesis: structure → philology → theology → sermon
  - Real original-language data, never from memory
  - Exact Strong's number matching, no homograph collisions
  - Tradition-aware, bias-honest

- **theology-claude-plugin** (fingerskier) — Exegetical theology research
  - 7 slash-command skills: historian, linguist, author, theologian, disciple, shepherd, research
  - Full exegetical document output
  - Confessional/evangelical tone

### Key Insight
**Biblical research AI tools prevent AI hallucination of Scripture** — they use real data, not model memory.
This is directly relevant to the user's Bible research project.

### Best Tool for Bible Research Accuracy
1. **claude-of-alexandria** — Most rigorous, 136 tests, TDD-built
2. **exegete** — Hallucination-resistant, real data, 4-stage exegesis
3. **ScriptureDeepDive** — 70+ scholars, accuracy auditor, CI pipeline
4. **studybible-mcp** — Fee & Stuart hermeneutics, lexicons
5. **theology-claude-plugin** — 7 skills, full exegetical output

### AI Clone & Marketing Tools — GitHub Scout

### AI Clone / Persona
- **MirrorMind** (12⭐, MIT) — Most complete: 7 agents, writing style profiling, GraphRAG, REST API, Telegram/Discord/WhatsApp, testing lab
- **CloneMe** (39⭐, MIT) — Lightweight digital twin, memory-driven, multi-platform, AI-provider agnostic
- **Real-Time-Voice-Cloning** (60K⭐, MIT) — Clone voice in 5 seconds, real-time
- **VoiceStudio/OmniVoice-Studio** (~18K⭐, AGPL-3.0) — Local ElevenLabs alternative, 646 languages, 4-8GB VRAM
- **ghost-writer** (7⭐, MIT) — 24-dimension forensic writing style clone
- **claude-email** (116⭐, MIT) — Full email marketing workflow with voice/persona matching

### Social Media
- **postiz-app** (36K⭐, AGPL-3.0) — 30+ platforms, AI copilot, MCP server
- **trypostit/trypost** (626⭐, AGPL-3.0) — 12 platforms, AI caption/hook generation

### Email Marketing
- **claros** (48⭐, MIT) — AI-native lifecycle email engine, deterministic plans
- **agentkits-marketing** (604⭐, MIT) — Enterprise AI marketing automation
- **email-marketing-bible** (301⭐, MIT) — 55K-word Claude Code skill, nonprofit playbook

### Key Gap
No single repo combines voice + writing style + social posting + email. Tov AI Clone must be assembled from components.
**VRAM reality:** Most repos are API-based (zero VRAM). Local LLM needs ~6-8 GB VRAM. Voice cloning ~4-6 GB GPU.

## Scholastic Research Tools — GitHub Scout

### RAG (Accuracy-First)
- **SciPhi-AI/R2R** (7,989⭐, MIT) — Production RAG, REST API, hybrid search, KG extraction
- **vignesh2027/VORTEXRAG** — 7-layer RAG, faithfulness 0.94 (vs 0.71 naive), causal drift filter
- **prabhaharanv/production-hybrid-rag** — Hybrid FAISS+BM25, cross-encoder rerank, citations

### Fact-Checking / Verification
- **Liyan06/MiniCheck** (219⭐, Apache-2.0) — EMNLP 2024, sentence-level fact-checking, Bespoke-MiniCheck-7B SOTA
- **QWED-AI/qwed-verification** (57⭐, Apache-2.0) — Deterministic verification, 100% error detection in benchmarks
- **sahilaf/FactEval** (4⭐, MIT) — Claim-level NLI verification, calibrated confidence

### Bible Study / Hermeneutics AI
- **djayatillake/studybible-mcp** — Fee & Stuart hermeneutics, Greek/Hebrew lexicons, MCP-native
- **asphaltsanai/exegete** — Hallucination-resistant 4-stage exegesis, real original-language data
- **davebream/claude-of-alexandria** — TDD-built, 136 automated tests, Christ-centered
- **ronanguilloux/skill-theo** — PaRDeS Jewish exegesis methods
- **Divine-Creative-Ministries/bible-cli** — Offline, clause-level syntax search, provenance tags

### Knowledge Graphs
- **FareedKhan-dev/agentic-knowledge-graph** — 929M edges, zero LLM calls, 83.2% PubMedQA
- **Agents4Academia-AI/prior** — Auditable claim graph from primary literature

### Key Gap
No tool combines midrashic/Haggadic hermeneutics with RAG verification. PaRDeS skill (skill-theo) is closest for Jewish exegesis but needs pairing with verification layer.

### Best for Bible Research Accuracy
1. **claude-of-alexandria** — Most rigorous, 136 tests, TDD-built
2. **exegete** — Hallucination-resistant, real data, 4-stage exegesis
3. **VORTEXRAG** — Best RAG faithfulness (0.94)
4. **MiniCheck** — Best fact-checking model (EMNLP 2024)
5. **studybible-mcp** — Fee & Stuart hermeneutics, lexicons

### Evidence Sources
- arxiv 2406.11717: Arditi et al. (refusal direction discovery)
- arxiv 2512.18901: Gabliteration
- arxiv 2308.10248: Turner et al.
- arxiv 2312.06681: Rimsky et al.

## MiniCPM5-2B — On-Device SOTA

### What It Is
- Dense 2B Transformer, 2,516,756,480 parameters
- 42 layers, 16 Q / 2 KV attention heads (GQA)
- 131,072 context length
- 2B-class open-source SOTA, average score 53.9
- Exceeds Qwen3.5-4B (51.1), Gemma-4-E2B, LFM2.5-2.6B

### Strengths
- Code reasoning, math reasoning, long-context understanding
- Tool use, agentic tasks
- RL + OPD post-training: +10.96 reasoning, +6.96 agentic
- 16 expert RL teachers distilled via On-Policy Distillation

### Deployment Options
| Backend | Format | Notes |
|---------|--------|-------|
| llama.cpp | GGUF | CPU/GPU, `-ngl 99` |
| vLLM | BF16/FP16 | OpenAI server |
| SGLang | BF16/FP16 | Recommended for tool calling |
| Ollama | GGUF | Local on-device |
| MLX | 4bit | Apple Silicon |
| LiteRT-LM | `.litertlm` | Android/iOS/desktop/IoT |

### MiniCPM-MoE-8x2B
- MoE variant: 8x2B experts
- Sparse attention + linear attention hybrid
- Million-token context support

### Relevance to GTX 1650
- 2B dense model fits on 4GB VRAM (1.8GB measured)
- Faster than Gemma-4-E2B (37 vs 33 tok/s with --reasoning off)
- Better coding/math than Gemma-4-E2B
- Supports speculative decoding (DSpark)
- **Produces working code with --reasoning off**

### Actual Test Results (GTX 1650, --reasoning off)
- Speed: 37 tok/s (20 tokens in 0.54s)
- Quality: Working Python code for LCS dynamic programming task
- VRAM: 1.8GB
- Output: Proper docstring, type hints, DP implementation

### Thinking Mode Fix — CRITICAL
**All models produce reasoning_content instead of code when thinking mode is on.**
Disable with `--reasoning off` in llama-server.

| Model | --reasoning off | Content Output | Speed |
|-------|-----------------|----------------|-------|
| MiniCPM5-2B | ✅ | Working Python code | 37 tok/s |
| Gemma-4-E2B | ✅ | Working Python + explanations | 37 tok/s |

### Pitfall: Thinking Mode (CRITICAL)
Thinking mode is on by default for Qwen3.5, Gemma-4, MiniCPM5-2B, and all other models on llama.cpp. It produces `reasoning_content` instead of usable code — the model always outputs a thinking chain, never direct code.

**Fix:** Always use `--reasoning off` in llama-server for coding tasks. Without this flag, no model on 4GB VRAM produces working code.

| Model | With thinking | With --reasoning off |
|-------|---------------|---------------------|
| MiniCPM5-2B | Empty content | Working Python code |
| Gemma-4-E2B | Empty content | Working Python code |
| Qwen3.5-4B | Empty content | Still thinking (unfixed) |
| Qwen2.5-0.5B | Empty content | Working Python code |
| Ornith-1.5-9B IQ2_M | Empty content | Working Python code |

If `--reasoning off` doesn't work, the model may use a different parameter name. Check the model's chat template for `enable_thinking` or `thinking` parameters.

### Install
```bash
# llama.cpp
llama-server -m MiniCPM5-2B-F16.gguf -ngl 99 -c 8192 --port 8080
# Ollama
ollama pull openbmb/MiniCPM5-2B
# vLLM
vllm serve openbmb/MiniCPM5-2B --port 8000
```
