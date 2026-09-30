# Securing Federated Intrusion Detection (CAIRLab Secure-AI Hackathon — Days 2–3)

Federated learning across 5 simulated banks on NSL-KDD, hardened against two failure modes:
**non-IID data skew** (Intermediate track) and a **poisoning attack by a malicious bank**
(Advanced track — our primary submission).

## Headline results (F1 on KDDTest+, mean over 3 seeds)

| Track | Setting | Naive FedAvg | Our method | Δ F1 |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | 1 poisoned bank (label-flip + update ×30), non-IID | **0.461** | **Multi-Krum 0.768** | **+0.308** |
| 🟡 Intermediate | non-IID (Dirichlet α=0.3) | 0.729 | **Balanced-FedProx 0.768** | **+0.039** |

- Malicious-bank **detection: precision = recall = 1.0**.
- Multi-Krum is **immune to attack magnitude** (flat F1 across ×10–×100) and **robust to an adaptive attacker** that evades norm-based filters.
- Final exported model (calibrated threshold baked in): **F1 = 0.895** on KDDTest+ at the 0.5 cutoff.

See [`WRITEUP.md`](WRITEUP.md) for the full technical report and [`media/`](media/) for figures.

## What's here

```
02_Intermediate_Advanced_Day2.ipynb   # full solution, run top-to-bottom with outputs
model_scripted.pt                      # final defended + calibrated model (TorchScript)
submission.json                        # all reported metrics (for automated verification)
WRITEUP.md                             # the report (< 1500 words)
VIDEO_SCRIPT.md                        # 3-minute walkthrough script
media/                                 # cover, strategy diagram, F1-over-rounds, scale sweep, confusion matrix
```

## Reproduce

Requirements: Python 3.10+, `torch scikit-learn pandas numpy matplotlib` (CPU is fine; < 10 min).

```bash
pip install torch scikit-learn pandas numpy matplotlib nbclient
jupyter nbconvert --to notebook --execute 02_Intermediate_Advanced_Day2.ipynb   # or run in Kaggle/Colab
```

The notebook downloads NSL-KDD automatically, rebuilds the seed-locked non-IID split, runs
every before/after comparison over 3 seeds, regenerates every figure in `media/`, and writes
`model_scripted.pt` and `submission.json`.

### Verify the primary number

```python
import torch, pandas as pd
# ...load & preprocess KDDTest+ exactly as in the notebook (Section 1) -> X_test, y_test
m = torch.jit.load("model_scripted.pt").eval()
preds = (torch.sigmoid(m(X_test)) > 0.5).int()
# F1(preds, y_test) == submission.json["self_reported_metrics"]["f1"]  (0.895)
```

## Method in one paragraph

**Intermediate:** the non-IID pain comes from *label* skew, so we rebalance each bank's local
loss (`pos_weight`) and add a FedProx proximal term to limit client drift — closing ~most of
the non-IID gap. **Advanced:** Multi-Krum selects the *n−f* most mutually-consistent client
updates by geometric distance and discards the outliers; because selection ignores magnitude,
the attacker's ×30 scaling backfires, and the discarded index is a perfect traitor detector.

## License

MIT.
