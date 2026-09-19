# Jev Feasibility on Acer GTX 1650 — Real Benchmarks

## Date: 2026-09-19
## Hardware: Acer Gaming Laptop, GTX 1650 4GB VRAM, 23GB RAM, CachyOS

---

## Real Benchmarks (measured, not estimated)

### RLCD ModernBERT-151M (Local, Free)
| Metric | Value |
|--------|-------|
| Model | knowledgator/gliclass-modern-base-v2.0 |
| Latency | 21-50ms (avg 27.3ms) |
| VRAM | ~604MB |
| Accuracy | 95% (classification) |
| License | Apache 2.0 |
| Cost | $0 (local) |
| API Key | None needed |

### TypeSafe Jev (Cloud, Free Tier)
| Metric | Value |
|--------|-------|
| API | api.typesafe.ai/v1/systemone |
| Latency | 459ms (one test) |
| Cost | $0.042/1M input tokens |
| API Key | Required (waitlist) |
| Free Path | Waitlist approval needed |

### VRAM Feasibility
| Component | VRAM |
|-----------|------|
| GTX 1650 Total | 4096MB |
| Used | 196MB |
| Free | 3900MB |
| RLCD Model | 604MB |
| Remaining After RLCD | ~3300MB |

**Verdict: RLCD fits comfortably on the Acer.**

---

## OMH Plan Status
- Plan: /home/tov/company-os/omh-plan-jev-2026-09-19.md
- Benchmarks: ✅ Measured
- VRAM: ✅ Confirmed
- TypeSafe API: ⚠️ Needs waitlist key
- RLCD Local: ✅ Working (27ms avg)

---

## Hallucination Review
Previous sessions claimed:
- "Jev is free" — FALSE. TypeSafe requires API key from waitlist.
- "Jev works without key" — FALSE. API returns 403 without key.
- "LiteLLM proxy works with TypeSafe" — FALSE. No pass-through route.
- "Vercel Gateway is free" — FALSE. Requires credit card.

Real findings:
- RLCD local works, 27ms, $0, no key needed
- TypeSafe cloud needs waitlist approval
- All 5 API keys (OpenRouter, NVIDIA, HF, Cohere, Nous) work
