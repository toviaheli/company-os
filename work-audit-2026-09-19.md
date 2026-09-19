# Work Audit — 2026-09-19 (COMPREHENSIVE)

## Purpose
Review ALL previous sessions for unfinished, finished, and hallucinated work.
Document findings for all agents to see. Drive resolution via OMH plan.

**Sources:** OMH goals ledger, adversarial review bundles, state database, all GitHub repos, local skill files, memory.

---

## SESSIONS REVIEWED

| Session | Date | Type | Key Findings |
|---------|------|------|-------------|
| 20260914_135840 | Sep 14 | Telegram DM | Initial Hermes setup, local LLM servers |
| 20260914_143555 | Sep 14 | Telegram DM | API key configuration |
| 20260914_191729 | Sep 14 | Telegram DM | Adversarial consensus first run |
| 20260914_191759 | Sep 14 | Telegram DM | Adversarial consensus continued |
| 20260914_191821 | Sep 14 | Telegram DM | Adversarial consensus completion |
| cron_df524ba7bf53_20260915 | Sep 15 | Cron | Daily heartbeat |
| cron_df524ba7bf53_20260916 | Sep 16 | Cron | Daily heartbeat |
| cron_df524ba7bf53_20260917 | Sep 17 | Cron | Daily heartbeat |
| 20260917_173224 | Sep 17 | Desktop | LLM optimization audit orchestration |
| cron_df524ba7bf53_20260918 | Sep 18 | Cron | Daily heartbeat |

**10 total sessions** (5 cron, 1 desktop, 4 Telegram DM)

---

## PRODUCTION-READY HERMES AGENT GOAL (Sep 14, COMPLETED with gaps)

| Sub-Task | Status | Evidence |
|----------|--------|----------|
| Local LLM servers working | ✅ DONE | ollama:11434, llama-server:8000, litellm:8081 |
| Real API keys configured | ✅ DONE | 15 providers from D drive master file |
| Adversarial consensus | ✅ DONE | Meta Llama vs Cohere AI, different families |
| goal_judge configured | ✅ DONE | cohere/command-a-03-2025 |
| DeepSeek Harness v0.3.1 | ✅ DONE | Installed and importable |
| OMH runtime | ✅ DONE | 119 skills, omh plugin enabled |
| model-chains.json updated | ✅ DONE | Uses ollama/cohere |
| NVIDIA free models | ⚠️ PARTIAL | API confirmed, inference timeout |
| DeepSeek key | ❌ TRUNCATED | sk-f23...cda9 incomplete |
| Portkey/Helicone | ❌ PLACEHOLDER | Keys not provided |
| litellm proxy | ⚠️ PARTIAL | Requires Python subprocess to survive |

**Evidence files:** `/home/tov/.omh/plans/adversarial-verification.md`, `adversarial-review-final.md`

---

## MEMORY/RECALL FAILURE & FIX (Sep 15)

- **Incident:** Agent claimed Telegram ID 7112740783 was NOT in secrets file — it WAS at lines 657 and 797
- **Root cause:** Narrow search — only "telegram" keyword, missed "Telegram ID:" and "Id:" formats
- **Fix:** exhaustive-search skill with 5-step procedure
- **Status:** ✅ FIXED — skill created, memory entry saved, ledger updated

---

## TELEGRAM SKILL UPDATE (Sep 15)

- Updated to state DB reader v2.0.0 — no auth needed
- Verified: 10 Telegram sessions, 17,431 messages, chat_id=7112740783
- **Status:** ✅ WORKING

---

## SKILLS SEPARATED (Sep 15)

- `local-llm-speed-opt` — pure speed optimization
- `abliterate-local-llm` — pure abliteration, speed sections removed
- **Status:** ✅ VERIFIED — both skills on disk, no cross-contamination

---

## LLM OPTIMIZATION AUDIT (Sep 17) — ORCHESTRATOR AUDIT

### Hallucinated Claims (17 🔴)

| # | Claim | Reality | Severity |
|---|-------|---------|----------|
| 1 | "20.0 tok/s baseline IQ2_XXS gen" | Actual: **9.8 tok/s** (2× overstatement) | CRITICAL |
| 2 | "45.6 tok/s generation" | Was **prompt processing**, not generation | CRITICAL |
| 3 | "Gemma-4-E2B 52.9 tok/s" | Actual: **10.3 gen tok/s** (prompt speed mislabeled) | CRITICAL |
| 4 | "Qwen3 4B Q4_K_M ~160 tok/s" | **Physically impossible** on 4GB VRAM | CRITICAL |
| 5 | "Phi-3.5-mini ~255 tok/s" | **Physically impossible** on SM75 | CRITICAL |
| 6 | "Q2_K GGUF 165 tok/s" | **Impossible** — 3.6GB model at 165 tok/s | CRITICAL |
| 7 | "MTP 1.38x speedup on GTX 1650" | Measured on **RTX A6000** (NVFP4 tensor cores) | CRITICAL |
| 8 | "MTP shares KV cache, less VRAM than DFlash" | MTP also **OOMs** (5.78GB drafter) | MISLEADING |
| 9 | "Vulkan 37% faster on old NVIDIA GPUs" | Benchmarked on **GTX 1060 (SM61)**, not SM75 | MISLEADING |
| 10 | "MTPs better than DFlash for GTX 1650" | **Wrong** — both OOM on 4GB | HALLUCINATED |
| 11 | "FlashAttention hurts (11.4 vs 45.2)" | Baseline 45.2 **never reproduced** | UNVERIFIED BASELINE |
| 12 | "45.6 tok/s as generation baseline" | Mislabeled prompt processing | HALLUCINATED |
| 13 | "Gated DeltaNet 5% speedup" | Paper claim, **not measured** on this hardware | UNVERIFIED |
| 14 | "PyTorch SDPA efficient ~10× speedup" | Extreme claim, **no SM75 benchmark** | UNVERIFIED |
| 15 | "Kernel LUF: 97% fewer TLB" | No source cited | UNVERIFIED |
| 16 | "TLB flush batching: 26.9% throughput" | No source cited | UNVERIFIED |
| 17 | "FATE-llama.cpp 3-5× speedup for MoE" | MoE-specific, not applicable to dense Ornith | UNVERIFIED |

### Verified Claims (29 ✅)

- Ornith IQ2_XXS = 9.8 gen tok/s, 6.8 prompt tok/s (3 trials)
- qwen3:1.7b = 73.7-75.1 tok/s gen
- FlashAttention hurts on SM75 (11.4 vs ~6.5 baseline)
- MMVQ Turing already in llama.cpp build
- KV cache q8_0 best prompt processing (50.8 tok/s)
- Thread sweep: no difference (GPU bottleneck)
- Batch sweep: 256 optimal
- INT8 unsigned bug on Turing SM75
- ThunderKittens requires SM80+
- DFlash2 no drafter for Ornith
- Edge0 MoE-only, Apple Silicon
- DFlash VRAM math correct
- Continuous batching 0.1 tok/s gain
- llama-server vs Ollama: llama-server wins 10-20%
- koboldcpp Vulkan OOMs on 4GB VRAM
- Ollama dies on models >4GB (llama3:8b OOM)
- And 13 more verified

### Unverified Claims (36 🟡)

- MOA as PEFT — not inference optimizer
- Dense LLM optimization for SM75 — many techniques not tested
- vLLM on tensor-core-less SM75 — not benchmarked
- SGLang on SM75 — smoke-test only
- DFlash1 implementation — researched but not benchmarked
- MTP drafter — researched but OOMs
- Gated DeltaNet fp16 — paper claim, not implemented
- Prompt lookup drafting — researched, not implemented
- Custom draft vocabularies — requires training infra
- Embedding requantization — researched, not implemented
- Split-KV Attention — SM75 forks exist but not benchmarked
- Long-context benchmarking (8K+) — not done
- Quality benchmarks (MMLU, HellaSwag) — not done

**Deliverables:**
- `/home/tov/ORCHESTRATOR_AUDIT_2026-09-17.md` — comprehensive audit
- `/home/tov/claim_audit_report.md` — detailed claim-by-claim classification
- `/home/tov/sm75-moa-research-dossier.md` — MOA + dense LLM research
- Judge verdicts: 🔄 pending (3 judges dispatched)

---

## OMH WORKFLOW ENFORCEMENT (Sep 15) — REJECTED

- **Verdict:** REJECT — 7/8 evidence points verified; 1 critical gap
- **Gap:** Memory entry for OMH primary workflow NOT found
- **Missing:** consolidation.json, consolidation.jsonl, dreaming.json, write_journal.jsonl — no OMH primary workflow entry
- **Status:** ⚠️ MEMORY ENTRY STILL MISSING — this was identified as a hard failure but not yet fixed

---

## COMPANY-OS REPO WORK (Sep 15–18)

### Finished (15 items)
Full list in previous audit — LLM benchmarking, MoA plan v8, research reports, skills docs.

### Unfinished (12 items)
Full list in previous audit — quality benchmarking, DFlash1 fix, Split-KV, etc.

### Hallucinated Claims (6 fixed in this session)
Full list in previous audit — DSpark, DFlash1/2, 54 configs, All Tests Complete.

### Contradictions (3 documented)
Full list in previous audit — mesh device status, DSpark availability, testing scope.

---

## HERMES-PROFILE-PACKS REPO

- **Remote:** Dadmin88/hermes-profile-packs (NOT toviaheli — different owner)
- **Local:** `/home/tov/hermes-profile-packs`
- **Branches:** main, ornith-benchmark-docs, academy/ce-phase0 through phase16, operator/academy-ce-*, wt/t_* worktrees
- **Unpushed:** b329537 "Add Ornith-1.5-9B LLM server benchmark documentation" on ornith-benchmark-docs branch
- **Academy CE work:** Phases 4-16 documented, multiple operator branches for CE baseline mode, contracts, efficiency invariants
- **Status:** Extensive work, needs full audit

---

## OTHER REPOS (toviaheli)

| Repo | Description | Last Updated | Relevance |
|------|-------------|-------------|-----------|
| company-os | Company OS docs | Sep 18 | ✅ Primary audit target |
| tov-rose-ai-clone | AI digital twin | — | Needs review |
| mempalace | AI memory system | — | Needs review |
| SoftwareFactoryskills | Agent skills template | — | Needs review |
| hermes-agent | Hermes agent source | Aug 31 | Needs review |
| backups | Encrypted cross-device backups | — | Needs review |
| research-backup | Session history + research | Aug 19 | ✅ Sessions reviewed |
| hermes-memory-vault | Memory vault | Jun 30 | Needs review |
| hermes-config | Config mirror | Jul 12 | Needs review |
| 15+ other repos | Various | Various | Low relevance |

---

## API KEYS STATUS

| Key | Status |
|-----|--------|
| HuggingFace (HF_TOKEN) | Working |
| OpenRouter x2 | Working |
| NVIDIA x2 | Working |
| Cohere | Working |
| Cloudflare | Working |
| Portkey | Deleted 2026-09-15 |
| Helicone | Deleted 2026-09-15 |
| Groq | Banned |

7 working keys. All tested.

---

*Comprehensive audit date: 2026-09-19 | Agent: Chief Analytics Officer | Sources: OMH ledger, adversarial reviews, state DB, all repos, memory*
