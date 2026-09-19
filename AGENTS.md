# AGENTS.md — company-os Repo Conventions

## Repo Purpose

This repo is the operating system for Tov's AI workflow. It holds:
- LLM optimization research and benchmarks (GTX 1650 4GB VRAM)
- MoA/MoE inference engine evaluations
- AI clone / marketing tool scouting
- Self-improving agent skills
- Work audits and OMH plans

## Structure

```
company-os/
├── AGENTS.md              ← This file: repo conventions
├── STATUS.md              ← Work status dashboard (auto-updated)
├── work-audit-*.md        ← Session audits (finished/unfinished/hallucinated)
├── omh-plan-*.md          ← OMH structured plans
├── moa-plan.md            ← MoA configuration plan (living doc)
├── moa-plan-review.md     ← Adversarial review of MoA plan
├── research-tracking-log.md ← Test results tracker (source of truth)
├── research-*.md          ← Research reports
├── ornith-benchmark-docs.md   ← Ornith server testing docs
├── ornith-server-testing.md   ← Ornith benchmark documentation
├── tov-ai-clone-research-report.md ← AI clone complete research
├── ai-clone-marketing-scout-report.md  ← GitHub scout
├── moe-inference-scout-report.md       ← MoE engine scout
├── mesh-moa-inference/          ← Mesh MoA skill
│   └── SKILL.md
└── self-improving-skills/       ← Self-improving skills
    └── SKILL.md
```

## Agent Rules

1. **Source of truth**: `research-tracking-log.md` for test results, `moa-plan.md` for config decisions
2. **No hallucinated claims**: Every numeric claim must be traceable to a test result. If untested, mark ❓ or "NOT DONE"
3. **Contradictions**: When two files disagree, check the tracking log first. If the tracking log is silent, flag as 🟡 contradiction in `work-audit-*.md`
4. **Hallucinated claims**: When found, fix the file AND document in `work-audit-*.md` with the correct value
5. **API keys**: Never commit keys. All 7 working keys are managed by Hermes. 100% failure if any key untested.
6. **Model deletion**: Flag only, never auto-delete. Archive/quarantine only.
7. **OMH workflow**: Plan first, then execute. Plan must have goals, non-goals, acceptance criteria, verification strategy.
8. **Commit messages**: Include date and brief description. Push after each logical unit of work.
9. **This repo is under `toviaheli/company-os`** on GitHub, not `c2rmarketing/company-os`.

## Key People

- **User**: Tov (Telegram chat ID: 7112740783)
- **Hermes profile**: agency-analytics-specialist
- **Hardware**: GTX 1650 Mobile 4GB, 23GB RAM, CachyOS
- **Company rule**: 100% failure if any key untested/failing

## Working API Keys (7)

| Provider | Status |
|----------|--------|
| HuggingFace (HF_TOKEN) | ✅ Working |
| OpenRouter x2 | ✅ Working |
| NVIDIA x2 | ✅ Working |
| Cohere | ✅ Working |
| Cloudflare | ✅ Working |

## Hallucination Watchlist

These claims were hallucinated in previous sessions and must NOT be repeated:
- ❌ "54 MoA configs tested" → actually ~14
- ❌ "DSpark ✅ for MiniCPM5-2B" → GGUF not found
- ❌ "DFlash1 BEST CANDIDATE" → actually slower than baseline (2.7 vs 20.0 tok/s)
- ❌ "DFlash2 best for old GPUs" → not available for Ornith
- ❌ "All Tests Complete — No Further Testing Needed" → contradicted by "Yet to Test" section
