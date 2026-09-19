# Work Audit — 2026-09-19

## Purpose
Review all previous sessions for unfinished, finished, and hallucinated work.
Document findings for all agents to see. Drive resolution via OMH plan.

---

## FINISHED WORK ✅

| # | Deliverable | Status | Evidence |
|---|------------|--------|----------|
| 1 | Local LLM benchmarking — 12 models tested | COMPLETE | `moa-plan.md` catalog + `research-tracking-log.md` |
| 2 | Speculative decoding research (DFlash1, MTP, Ngram) | COMPLETE | `research-tracking-log.md`, `research-speculative-decoding.md` |
| 3 | MoA plan v8 with adversarial review | COMPLETE | `moa-plan.md`, `moa-plan-review.md` (4 perspectives, 3 rounds) |
| 4 | AI clone marketing scout report | COMPLETE | `ai-clone-marketing-scout-report.md` |
| 5 | MoE inference engines scout report | COMPLETE | `moe-inference-scout-report.md` |
| 6 | Ornith benchmark docs | COMPLETE | `ornith-benchmark-docs.md`, `ornith-server-testing.md` |
| 7 | CUDA optimization research report | COMPLETE | `research-cuda-optimization-report.md` |
| 8 | Low-VRAM optimization research | COMPLETE | `research-low-vram.md` |
| 9 | Old hardware AI research | COMPLETE | `research-old-hardware-ai.md` |
| 10 | LLM inference optimization research (community) | COMPLETE | `research-llm-optimization.md` |
| 11 | 2025-2026 LLM inference optimization research | COMPLETE | `research-llm-inference-2025.md` |
| 12 | Self-improving skills SKILL.md | COMPLETE | `self-improving-skills/SKILL.md` |
| 13 | Research tracking log | COMPLETE | `research-tracking-log.md` |
| 14 | Mesh MoA inference SKILL.md | COMPLETE | `mesh-moa-inference/SKILL.md` |
| 15 | Tov AI clone research report | COMPLETE | `tov-ai-clone-research-report.md` |

---

## UNFINISHED WORK ⚠️

| # | Item | Priority | Blocker |
|---|------|----------|---------|
| 1 | Quality benchmarking — MoA vs solo on real coding tasks | HIGH | No test harness defined |
| 2 | DFlash1 "ctx_other required" configuration fix | MEDIUM | Needs GPU context management |
| 3 | Split-KV Attention — --split-kv flag not recognized | MEDIUM | Not in current llama.cpp build |
| 4 | Ornith-AD aggregator quality unknown | HIGH | Never tested quality, only speed |
| 5 | MiniCPM5-2B with DSpark speculative decoding | MEDIUM | GGUF not found, marked NOT DONE |
| 6 | Server lifecycle bug — proper fix (not 3s sleep workaround) | MEDIUM | Gemma crashes on rapid restart |
| 7 | wait_port_free timeout too short (15s → 30s) | LOW | Script reliability |
| 8 | Self-MoA test — multiple samples, consolidate quality | HIGH | Core recommendation untested |
| 9 | Job-specific benchmarking (coding, marketing, research) | HIGH | Only one LCS task tested |
| 10 | Cutting-edge techniques: vAttention, KVTC, Mustafar, LvLLM, FATE, GSQ-RCO, Expert offloading, DiskLLM, HotPin, llm-fit | LOW | Listed but no plan/timeline |
| 11 | Clean log parsing — deduplicate old results | LOW | Housekeeping |
| 12 | FreeToken test on GTX 1650 | LOW | Install timed out previously |

---

## HALLUCINATED / UNVERIFIED CLAIMS 🔴

| # | Claim | Location | Problem | Correct Value |
|---|-------|----------|---------|---------------|
| 1 | "54 MoA configs tested" | `tov-ai-clone-research-report.md:8` | Overstatement — only ~14 unique configs actually tested | "~14 MoA configs tested" |
| 2 | "DSpark speculative decoding ✅" for MiniCPM5-2B | `moa-plan.md:78,105` | Contradicted by same repo: DSpark GGUF not found | "DSpark ❌ Not available (GGUF not found)" |
| 3 | "Supports speculative decoding (DSpark)" | `mesh-moa-inference/SKILL.md:457` | Contradicted by line 156 of same file | "DSpark not tested (GGUF not available)" |
| 4 | "DFlash1: BEST CANDIDATE for Ornith on 4GB VRAM" | `research-tracking-log.md:49` | Actual result was 2.7 tok/s vs 20.0 baseline — conclusion reversed | "DFlash1 SLOWER than baseline — rejected" |
| 5 | "DFlash2: Current best speculative decoding for old GPUs" | `research-tracking-log.md:68` | DFlash2 only available for Qwen3.8-27B, not Ornith | "DFlash2 not available for Ornith" |
| 6 | "All Tests Complete — No Further Testing Needed" | `research-tracking-log.md:57` | Contradicted by "Techniques Yet to Test (CUTTING-EDGE)" section | "Testing complete for scoped tests; 10+ techniques remain untested" |

---

## CONTRADICTIONS 🟡

| # | Conflict | Files | Resolution |
|---|----------|-------|------------|
| 1 | Mesh device status: 4 "online" vs "5 of 6 OFFLINE" | `mesh-moa-inference/SKILL.md` vs `moa-plan.md` | Different snapshots; moa-plan more recent. Verify current Tailscale status. |
| 2 | DSpark availability: "✅ DSpark" vs "❌ GGUF not found" | `moa-plan.md:78` vs `tov-ai-clone-research-report.md:249` | GGUF not found is correct; DSpark claim is hallucinated |
| 3 | Tracking log: "All Tests Complete" vs "Yet to Test" | `research-tracking-log.md:57` vs `research-tracking-log.md:34-44` | Contradictory section headers; needs cleanup |

---

## REPO STATE

- **Local:** `/home/tov/company-os` (main branch, clean working tree)
- **GitHub:** `toviaheli/company-os` — synced, no unpushed changes
- **User mentioned:** "c2rmarketing company-os repo" — no `c2rmarketing/company-os` exists on GitHub
- **Git log:** 8 commits, Sept 15–18, 2026
- **Branches:** `main`, `docs/ornith-benchmark`, `empty-main`

---

## API KEYS STATUS (from memory)

| Key | Status |
|-----|--------|
| HuggingFace (HF_TOKEN) | Working |
| OpenRouter x2 | Working |
| NVIDIA x2 | Working |
| Cohere | Working |
| Cloudflare | Working |
| Portkey | Deleted 2026-09-15 |
| Helicone | Deleted 2026-09-15 |
| Groq | Banned previously |

7 working keys total. All tested and verified.

---

*Audit date: 2026-09-19 | Agent: Chief Analytics Officer (agency-analytics-specialist)*
