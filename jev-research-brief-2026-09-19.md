# Jev (TypeSafe AI) — OMH Research Brief

## OMH Run: 20260919T004009Z-web-research-jev

**Retrieval date:** 2026-09-19
**Sources:** typesafe.ai blog, docs.typesafe.ai, Arize AI, orcarouter.ai, meetcody.ai, developersdigest.tech, lilting.ch, pearpages.com, theregister.com, Vercel AI Gateway
**Confidence:** HIGH (multiple independent sources confirm pricing; vendor claims marked separately)

---

## What Jev Actually Is

TypeSafe AI's System One model, launched Sep 15, 2026 by Diogo Almeida (InstructGPT co-author).

**It does NOT generate text.** It takes structured state + typed questions, returns typed answers with probabilities in a single parallel forward pass.

| Primitive | What it does | Example |
|-----------|-------------|---------|
| Choice | Pick one option from list | "Which worker next?" |
| Score | Evaluate against scale | "How relevant is this source?" (0-2) |
| Noul | Yes/no probability | "Should this be published?" |

---

## Real Pricing (vendor-confirmed)

| Metric | Jev | GPT-5.6 Terra | Claude Opus 5 |
|--------|-----|---------------|---------------|
| Input | $0.042/M tokens | $2.00/M tokens | ~$0.30/M tokens |
| Output | FREE | $12.00/M tokens | ~$15.00/M tokens |
| Latency | 70-500ms | 10-38s | 38-92s |
| Cost/case | $0.0004 | $0.0304 | $0.1761 |

**10,000 decisions = $0.42** (vs $304 for Terra, $1,761 for Opus 5)

---

## Real Benchmarks (from web sources)

### TypeSafe's Own Eval (4 workflows, vendor-run)
| Model | Accuracy | Cost/case | Latency |
|-------|----------|-----------|---------|
| Jev | 67.8% | $0.0004 | 0.4s |
| GPT-5.6 Terra | 67.9% | $0.0304 | 10.1s |
| GPT-5.6 Sol | 74.1% | $0.0836 | 23.3s |
| Claude Opus 5 | 73.1% | $0.1761 | 37.8s |
| Claude Sonnet 5 | 67.8% | $0.1174 | 78.1s |

**Caveat:** Reference answers = average of GPT-6 Astra + Claude Fable 5.1, NOT human ground truth. No independent reproduction yet.

### Independent Test (Every/Malte Ubl)
- 777 judgments in <0.7s for ~$0.0025
- 1,709 judgments for <$0.01
- 25x faster, 580x cheaper than Claude Fable 5.1
- Caught 6/7 planted defects (Fable caught all 7)

### Doom Demo
- 10 calls/sec, ~$7/hour
- Jev steers game bot via text state description

### Paper Classification
- 1,018 AI papers for $0.08
- Median 256ms per paper
- 98.3% accuracy (zero-shot, no fine-tuning)

---

## Feasibility on Gaming Acer

### Acer Specs (observed)
| Component | Spec |
|-----------|------|
| CPU | Intel i5-9300H @ 2.40GHz (4C/8T) |
| GPU | GTX 1650 Mobile, 4GB VRAM |
| RAM | 23GB total, 13GB available |
| Disk | 235GB total, 29GB free (⚠️ 88% full) |

### Can Jev Run on This Acer?

**Jev itself is a cloud API** — it runs on TypeSafe's servers, not locally. The Acer just needs:
1. Python 3.12+ ✅ (can install)
2. Internet connection ✅ (has it)
3. ~200MB for Python venv + SDK ✅ (disk space tight but OK)

**Browser Use + Jev** can run on this Acer:
- Browser Use: Python + Chromium browser ✅
- Jev client: typesafe-sdk via pip ✅
- API calls: US West Coast latency (~40-80ms from US)

**Local LLM fallback** on GTX 1650 4GB:
- MiniCPM5-2B: 37 tok/s ✅
- Gemma-4-E2B: 37 tok/s ✅
- Qwen2.5-0.5B: 72 tok/s ✅
- Ornith-1.5-9B IQ2_M: 13 tok/s ⚠️ slow

### Realistic Use Case for This Acer

| Scenario | Jev Cost | Browser Use Cost | Total |
|----------|----------|------------------|-------|
| 100 research tasks/day | $0.042 | ~$0.50 | ~$0.54/day |
| 1,000 tasks/month | $4.20 | ~$15 | ~$19/month |
| 10,000 tasks/month | $42 | ~$150 | ~$192/month |

**Browser Use costs** are separate (browser infrastructure, not Jev).

---

## Caveats (from sources)

1. **Early access waitlist** — not generally available yet
2. **No paper/independent benchmark** — vendor runs its own evals
3. **Accuracy is mid-tier** — 67.8% vs 74% for best LLMs on TypeSafe's eval
4. **Input only text** — no image/audio at launch
5. **255 choice max** — larger lists need two-stage scoring
6. **Pricing may be subsidized** — TypeSafe says "early pricing may be subsidized"
7. **No free tier** — API key required, no credit mentioned
8. **Not a chatbot** — cannot write, code, or explain
9. **Disk space** — Acer has only 29GB free (88% full)

---

## Honest Assessment

**Jev is real but overhyped.** The 193.6x faster / 444.6x cheaper claims are vendor-best-case, measured on TypeSafe's own workflows against the slowest baselines. Independent test shows ~25x faster, ~580x cheaper on a small sample.

**What's genuine:**
- $0.042/M input tokens is real and cheap
- 70-500ms latency is real
- Typed decisions with probabilities are genuine
- Zero hallucination (by construction — no text generation)

**What's exaggerated:**
- "Frontier intelligence" — it's mid-tier accuracy
- "193.6x faster" — best-case only, not universal
- "Can't hallucinate" — technically true but can still be wrong
- The Doom demo is a toy, not production evidence

**For this Acer:**
- Jev API calls work from anywhere with internet ✅
- Browser Use runs locally ✅
- GTX 1650 4GB is fine for Browser Use ✅
- Disk space is tight (29GB free) ⚠️
- RAM is fine (13GB available) ✅
- Real cost: ~$0.54/day for 100 tasks ⚠️ (depends on usage)

---

## Sources

1. https://typesafe.ai/blog/introducing-system-one-models-and-jev (vendor launch)
2. https://docs.typesafe.ai/models (official docs)
3. https://flaviocopes.com/jev/ (independent deep dive)
4. https://arize.com/blog/typesafe-jev-llm-judge/ (Arize AI analysis)
5. https://www.orcarouter.ai/blog/jev-typesafe-system-one-what-we-know (independent analysis)
6. https://meetcody.ai/blog/typesafe-jev-ai-system-one-model/ (independent analysis)
7. https://developersdigest.tech/blog/typesafe-jev-system-one-models-release-guide-2026 (independent)
8. https://pearpages.com/blog/2026/09/16/jev-sorted/ (independent analysis)
9. https://www.theregister.com/ai-and-ml/2026/09/16/typesafe-ai-debuts-model-for-machines-that-plays-doom/ (The Register)
10. https://custom.typingmind.com/tools/estimate-llm-usage-costs/vercel/jev (Vercel pricing)

---

*OMH research brief | retrieved 2026-09-19 | Chief Analytics Officer*
