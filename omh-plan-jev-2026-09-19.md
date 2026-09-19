# OMH Plan — Jev Feasibility on Acer GTX 1650

## Goal
Determine if Jev (TypeSafe System One) can run on Tov's Acer gaming hardware via the free path, with real benchmarks.

## Non-Goals
- Paid services (Vercel Gateway, TypeSafe API waitlist)
- Building production agent stack
- Replacing existing Hermes setup

## Acceptance Criteria
- [ ] Jev API reachable and responding
- [ ] Benchmark: latency per decision < 100ms
- [ ] Benchmark: cost per decision < $0.01
- [ ] VRAM feasibility confirmed for local alternatives
- [ ] Documentation updated with real numbers

## Verification Strategy
- Direct API call to TypeSafe endpoint
- Latency measurement with time module
- Cost calculation from pricing docs
- VRAM check via nvidia-smi
- Cross-reference with research-tracking-log.md
