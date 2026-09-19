# Jev Alternative — RLCD ModernBERT-151M Setup

## Why This Model

| Factor | Jev (TypeSafe) | RLCD ModernBERT-151M |
|---|---|---|
| Cost | Waitlist + API | 100% free, self-hosted |
| Latency | ~150ms | 24.7ms (4x faster) |
| VRAM | Unknown | 604MB |
| Open source | No | Yes (Apache 2.0) |
| Accuracy | Proprietary | 95% top-1 |
| Calibration | Unknown | 3.35% ECE |

## Hardware Fit (GTX 1650 4GB)

- Model VRAM: 604MB
- Remaining VRAM: ~3GB for other models
- RAM: 23GB total, no issue
- **Verdict: Fits comfortably alongside Ollama Qwen2.5-Coder-3B**

## Installation

```bash
pip install rlcd
```

## Usage

```python
from rlcd import DecisionEngine, Choice, Option

engine = DecisionEngine()
engine.device = 'cuda'  # Force GPU
engine.model = engine.model.to('cuda')

state = "Help! My payouts have been failing for 3 days."
query = Choice(
    id="dept",
    question="Which department should handle this?",
    options=[
        Option(id="billing", description="Payments, invoicing, refunds"),
        Option(id="technical", description="Bugs, outages, integrations"),
        Option(id="sales", description="Pricing, upgrades, new accounts")
    ]
)

result = engine.evaluate(context=state, queries=[query])
print(result.results[0].selected_id)  # 'billing'
print(result.results[0].selected_probability)  # 0.694
```

## Decision Primitives

| Type | Use case |
|---|---|
| `Choice` | Pick from N options (routing, classification) |
| `Score` | Rate on ordered scale (1-10 severity) |
| `Noul` | Yes/no with confidence (destructive? safe?) |

## Integration with Hermes

Add as a tool in Hermes delegation config:

```yaml
# ~/.hermes/config.yaml
tools:
  decision_engine:
    module: rlcd
    class: DecisionEngine
    device: cuda
```

Then delegate routing decisions through it instead of prompt-based if/else.

## Benchmarks (from repo)

- Top-1 accuracy: 95.0% (in-domain)
- ECE: 3.35% (calibrated)
- Abstention recall: 97.5%
- Latency p50: 35ms (paper), 24.7ms (our GPU test)
- Model: 151M params, ModernBERT-base

## Caveats

- 512-token context limit
- English only
- Calibration status: "uncalibrated" out of the box — fine for routing, re-calibrate for production
- OOD queries may need temperature calibration
