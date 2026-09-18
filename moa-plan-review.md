# Adversarial Review: GTX 1650 MoA Configuration Plan

## Roster
1. **Skeptic** — challenges assumptions, finds flaws
2. **Validator** — confirms what works, tests evidence
3. **Engineer** — practical implementation concerns
4. **Researcher** — quality and accuracy focus

## Round 1: Independent Findings (blind)

### Skeptic
- The plan claims "aggregator determines speed, not proposer pair" but only tested 5 models. What about other models?
- Ornith-AD is IQ2_XXS-IQ1_M quant — extremely low quality. 20 tok/s is meaningless if output is garbage.
- The plan recommends Qwen+Gemma → Ornith-AD but never tested this specific combo end-to-end with actual work.
- Self-consistency voting (3 rounds) adds 3x latency for no measured quality improvement.

### Validator
- Confirmed: all 5 models start individually and produce valid output.
- Confirmed: Ornith-AD aggregator consistently gives 20.0 tok/s across all proposer pairs and temperatures.
- Confirmed: temperature has no meaningful effect on speed (variance < 0.5 tok/s).
- Confirmed: Gemma-4-E2B solo at 56 tok/s is faster than any MoA config.

### Engineer
- The server lifecycle bug (Gemma crashes on rapid restart) is NOT fixed in the script — only worked around by adding 3s sleep between servers.
- The moa-finish.py script uses `wait_port_free` which blocks indefinitely if the port never frees.
- No error handling for partial failures — if one config fails, the whole script continues but results are incomplete.
- The plan recommends sequential server loading but doesn't address the startup overhead (3x per round = ~15s overhead per config).

### Researcher
- The plan cites community sources (Localmaxxing, FitMyLLM) but never verified their claims against actual test data.
- No comparison of MoA quality vs solo model quality — speed is measured but not accuracy.
- The plan assumes Ornith-AD is the best aggregator but never tested quality, only speed.
- No benchmark against the claimed "best possible configuration" — the plan is untested on actual jobs.

## Round 2: Cross-Attack

### Skeptic attacks Validator
- "Confirmed" findings are just repeat measurements of the same config. You haven't tested novelty.
- The 20 tok/s number is an average of 3 rounds — what's the variance? If it's ±5 tok/s, it's not reliable.

### Engineer attacks Skeptic
- The server lifecycle bug IS partially fixed — the 3s sleep prevents most crashes. But it's not a real fix.
- The `wait_port_free` function has a 15s timeout which is too short for slow GPUs. Should be 30s.

### Researcher attacks Validator
- Speed is not the same as quality. A 20 tok/s aggregator that produces garbage is worse than a 3 tok/s one that produces good output.
- The plan never measured MoA quality vs solo model quality. This is a critical gap.

### Validator attacks Researcher
- The plan explicitly states speed is the primary metric and quality is secondary for this hardware constraint.
- The plan acknowledges Ornith-AD is a low-quality quant and recommends it only for speed.

## Round 3: Verdicts

### Skeptic → Skeptic's own findings
- **Concede**: The plan does acknowledge limitations (Ornith-AD quality, 5-model test set).
- **Defend**: The plan should NOT recommend MoA for quality-critical tasks without quality benchmarks.
- **Refine**: Add quality testing as a separate phase before recommending MoA for production use.

### Validator → Skeptic's findings
- **Defend**: The 20 tok/s number is reliable (variance < 0.3 tok/s across 3 rounds).
- **Refine**: Add variance reporting to all speed measurements.

### Engineer → Engineer's own findings
- **Concede**: The server lifecycle bug is not fully fixed — the 3s sleep is a workaround, not a fix.
- **Defend**: The `wait_port_free` timeout should be increased to 30s for reliability.

### Researcher → Researcher's own findings
- **Concede**: Quality benchmarking is a critical gap that needs to be addressed.
- **Defend**: Speed benchmarking is valid for hardware evaluation but insufficient for production recommendations.

## Distilled Bundle

### Hard Constraints
1. GTX 1650 has 4GB VRAM — only one model can run at a time
2. Ornith-AD aggregator is IQ2_XXS-IQ1_M quant — speed is 20 tok/s but quality is unknown
3. Gemma-4-E2B server crashes if started within 3s of another server
4. MoA requires sequential server loading (3x startup overhead per round)

### Decisions
1. **Primary config**: Qwen3.5-4B + Gemma-4-E2B → Ornith-AD @ -ngl 99, temp=0.3 (20 tok/s)
2. **Fallback (speed-critical)**: Gemma-4-E2B solo at 56 tok/s
3. **Quality testing**: Required before production use of MoA for any job

### Risks
1. Ornith-AD quality may be too low for production use
2. Server lifecycle bug may cause intermittent failures
3. No quality benchmarking for MoA vs solo models
4. GTX 1650 bandwidth (128 GB/s) limits all models

### Open Questions
1. What is the quality of Ornith-AD aggregator output vs solo Gemma-4-E2B?
2. Does MoA improve quality enough to justify the 3x speed penalty?
3. Should we test other aggregators (e.g., Qwen3.5-4B as aggregator)?
4. Is 20 tok/s sufficient for interactive coding tasks?

## Mandatory Handoff
This bundle is INPUT to planning, not the plan itself. Hand to `plan` workflow for the detailed plan with acceptance criteria and verification strategy.
