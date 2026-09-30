# Securing Federated Intrusion Detection (CAIRLab Secure-AI Hackathon — Days 2–3)

Federated learning across 5 simulated banks on NSL-KDD, hardened against two failure modes:
**non-IID data skew** (Intermediate track) and a **poisoning attack by a malicious bank**
(Advanced track — our primary submission). Team: **Abugida**.

## Headline results (F1 on KDDTest+, mean over 3 seeds)

| Track | Setting | Naive FedAvg | Our method | Δ F1 |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | largest bank (client 2, ~43% of data) poisoned: label-flip + update ×20, non-IID | **0.211** | **Multi-Krum 0.780** | **+0.569** |
| 🟡 Intermediate | non-IID (Dirichlet α=0.3) | 0.729 | **SCAFFOLD + class-balanced 0.797** | **+0.068** |

- Malicious-bank **detection: precision = recall = 1.0**.
- Multi-Krum is **immune to attack magnitude** (flat ≈0.78 across ×10–×100 while naive collapses).
- **Attacker's dilemma:** an adaptive attacker that caps its update to evade norm-based filters
  becomes too weak to matter (naive only 0.76→0.73), while a loud attack is detected & undone —
  Multi-Krum lands at ≈0.78 either way; norm-based detection is blinded (recall 0.0) *and* harmful.
- **Defense-in-depth:** even with **2 of 5 banks** compromised (beyond Krum's formal n=5 limit),
  coordinate-median recovers to **0.813** and Multi-Krum (f=2) to **0.794** (detection recall 1.0).
- Final exported model: **0.783 F1 @0.5** (honest), **0.906** with a calibrated threshold baked in.

See [`WRITEUP.md`](WRITEUP.md) for the full report and [`media/`](media/) for figures.

## What's here

```
02_Intermediate_Advanced_Day2.ipynb   # full solution, run top-to-bottom with outputs
model_scripted.pt                      # final defended + calibrated model (TorchScript)
submission.json                        # all reported metrics (for automated verification)
WRITEUP.md                             # the report (< 1500 words)
VIDEO_SCRIPT.md                        # 3-minute walkthrough script
media/                                 # cover, strategy diagram, F1-over-rounds, scale sweep,
                                       #   attacker-dilemma, multi-attacker, confusion matrix
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
import torch
# ...load & preprocess KDDTest+ exactly as in the notebook (Section 1) -> X_test, y_test
m = torch.jit.load("model_scripted.pt").eval()
preds = (torch.sigmoid(m(X_test)) > 0.5).int()
# F1(preds, y_test) == submission.json["self_reported_metrics"]["f1"]  (0.906)
```

## Method in one paragraph

**Intermediate:** the non-IID pain has two sources, so we fix both — SCAFFOLD control variates
cancel client *drift*, and a class-balanced local loss neutralises *label* skew — overshooting
the IID ceiling. **Advanced:** Multi-Krum keeps the *n−f* most mutually-consistent client updates
by geometric distance and discards the outliers; because selection ignores magnitude, an
attacker's scaling backfires, and the discarded index is a perfect traitor detector. We
stress-test it with an adaptive norm-mimicking attacker (the attacker cannot win either way) and
with two simultaneous traitors (graceful degradation).

## License

MIT.
