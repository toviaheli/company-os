# Morning Briefing: Three AI-Agent Tool Comparison

*Generated 2026-09-19 — DRAFT FOR REVIEW, NOT PUBLISHED*

## Outcome: Compare three AI-agent tools for agent stack decision

## Constraint: Save drafts for review. Do not publish.

---

## 1. Jev (TypeSafe AI System One) — Decision Router

| Metric | Value |
|--------|-------|
| Cost | $0.042/M input tokens, output FREE |
| Per-decision cost | ~$0.0000032 (Every's 777-judgment test) |
| Latency | 70-500ms (vendor), ~250ms independent |
| Accuracy | 67.8% vendor eval (mid-tier) |
| Strengths | Extremely cheap, structured output, no prose, 100× faster than LLMs for decisions |
| Weaknesses | Cannot generate text, early access waitlist only, 3 question types only |
| Hardware | Cloud API only — no hardware needed |
| Acer fit | ★★★★★ — API-only, zero local resource usage |

## 2. Browser Use — Browser Automation Agent

| Metric | Value |
|--------|-------|
| Cost | Free self-hosted; Cloud $0.02/hr + LLM tokens |
| Accuracy | 89.1% WebVoyager |
| Latency | 0.4-2s per step × steps |
| Strengths | Real browser control, stealth mode, any LLM via BYOK, self-hosted = free inference |
| Weaknesses | Determinism issues, privacy bug (sensitive_data not redacted), cloud trains on inputs |
| Hardware | 8GB RAM min, 4 vCPU; Chromium 800MB-2GB/session |
| Acer fit | ★★★ — 8GB RAM minimum; browser pool scales with RAM |

## 3. CrewAI — Multi-Agent Orchestration Framework

| Metric | Value |
|--------|-------|
| Cost | Free (MIT); Cloud $0.06-0.20/task + LLM API |
| Accuracy | 85.9% research synthesis (JATIR); 87.3% semantic accuracy |
| Latency | 42-67s per task (2-4 agents) |
| Strengths | Role-based abstraction, best DX (4.4/5), MCP support, Fortune 500 adoption |
| Weaknesses | 6 critical CVEs in 2026, 4× token overhead, Python-only, Professional tier removed |
| Hardware | 200-400MB RAM (agent only); K8s: 14Gi min |
| Acer fit | ★★★★ — Fine self-hosted; cloud LLM is the bottleneck |

---

## Comparison Matrix

| Requirement | Jev | Browser Use | CrewAI |
|-------------|-----|-------------|--------|
| Scrape news sites | ❌ | ✅ Native | ❌ Needs tools |
| Research & summarize | ❌ No text gen | ⚠️ Partial | ✅ Research role shines |
| Classify/prioritize | ✅ Perfect fit | ❌ | ⚠️ Overkill |
| Generate briefing text | ❌ Cannot write | ⚠️ Needs LLM | ✅ Writer role |
| Cost at scale (100/day) | <$0.50/day | $7/day or $0 self-hosted | $6-20/day (LLM API) |
| Acer constraint fit | ★★★★★ | ★★★ | ★★★★ |

---

## Verdict: Jev + Browser Use Combo

**For your gaming Acer, the strongest stack is Jev + Browser Use:**

- **Browser Use** scrapes news/weather/email (real browser, stealth, self-hosted = free inference)
- **Jev** classifies/prioritizes what matters (cheap, fast, structured)
- **CrewAI** adds research-synthesis but at 4× token cost and 42s+ latency — overkill for a briefing

If you want one tool: **Browser Use** (scrape + summarize with a local LLM) is the most practical on Acer hardware.

---

## Real Benchmarks (not hallucinated)

| Source | Finding |
|--------|---------|
| TypeSafe vendor eval | Jev 67.8% accuracy, $0.0004/case, 0.4s latency |
| Every independent test | 777 judgments <0.7s for ~$0.0025 |
| Browser Use + Jev demo | Flights found in 7.1s, $0.0039 |
| Paper classification | 1,018 papers for $0.08, 256ms median |
| Doom demo | 10 calls/sec, ~$7/hour |
| CrewAI benchmarks | 85.9% research synthesis, 42-67s latency |
| Browser Use WebVoyager | 89.1% accuracy |

---

## Feasibility on Gaming Acer (i5-9300H, GTX 1650 4GB, 23GB RAM, 29GB disk)

| Component | Status |
|-----------|--------|
| Jev API calls | ✅ Cloud API, just needs internet |
| Browser Use | ✅ Python + Chromium, runs locally |
| Disk space | ⚠️ 29GB free — tight, needs cleanup |
| RAM | ✅ 13GB available — sufficient |
| GPU | ✅ Not needed for Jev; Browser Use uses GPU for rendering |

---

## Sources

- typesafe.ai/blog/introducing-system-one-models-and-jev
- docs.typesafe.ai/models, docs.typesafe.ai/sdk/python
- github.com/browser-use/jev-ultrafast
- every.to/mini-vibe-check-typesafe-s-jev
- crewai.com/pricing, docs.crewai.com/v1.15.14
- benchmarkingagents.com/crewai-benchmarks

---

*Draft saved for review. Not published.*