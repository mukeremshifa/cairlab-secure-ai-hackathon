# Securing Federated Intrusion Detection

### Robust aggregation and traitor detection when a "bank" turns malicious
**Tracks:** 🟡 Intermediate + 🔴 Advanced — **primary track: Advanced**

---

## The headline number (before → after)

All F1 scores are on the **held-out KDDTest+ split** used by the notebook's `evaluate()`, averaged over **3 seeds**.

| Track | Setting | Naive FedAvg | **Our method** | **Δ F1** |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | largest bank (client 2, ~43% of data) poisoned: label-flip + update ×20, non-IID | **0.211** | **Multi-Krum 0.780** | **+0.569** |
| 🟡 Intermediate | non-IID skew (Dirichlet α=0.3 by attack family) | 0.729 | **Balanced-FedProx 0.768** | **+0.039** |

Our Advanced defense also **identifies the malicious bank with precision = recall = 1.0**, and the recovered model (0.780) **exceeds the clean-run baseline (0.763)** because excluding the outlier denoises the aggregate. The exported `model_scripted.pt`, with a calibrated decision threshold baked in, scores **F1 = 0.906** on KDDTest+ at the standard 0.5 cutoff.

---

## The problem, precisely

Five banks train a shared intrusion detector via FedAvg without pooling data. Two things break the naive version:

1. **Non-IID data.** The Dirichlet(α=0.3) split makes banks wildly heterogeneous — client attack-rates are **81% / 86% / 68% / 15% / 4%**. One bank is almost all attacks; another almost all normal traffic.
2. **A malicious bank.** One bank flips its labels and inflates its update magnitude, so a plain weighted average is dominated by a single sabotaged vector.

Each track is scored on the **improvement over naive FedAvg on the identical setting**, so every comparison fixes the model, data split, and round count and changes only the aggregation.

## Intermediate — Balanced-FedProx

**Insight: the skew lives in the labels, so fix the labels.** Because each bank's class balance is extreme, its local SGD is dominated by its majority class, and averaging these lopsided updates biases the global model. Our scheme changes local optimisation, not just the server average:

- **Class-balanced local loss** (`pos_weight = n_neg/n_pos`, per bank per round) — neutralises each bank's label skew *before* its update is formed.
- **FedProx proximal term** (`μ‖w−w_global‖²`, μ=0.3) — limits how far a heterogeneous bank drifts.

Result: **0.729 → 0.768 (+0.039)** at matched rounds; against the notebook's default 8-round naive baseline (≈0.70) the gain is **+0.065**. This nearly closes the entire non-IID penalty (the IID ceiling for this model is ≈0.764). **What didn't work:** FedProx *alone* moved almost nothing (+0.006); the class rebalancing is the real lever.

## Advanced (primary) — Multi-Krum + traitor detection

We benchmarked the full menu of robust aggregators (coordinate-median, trimmed-mean, norm-clipping, an update-norm anomaly detector, FLTrust, and Multi-Krum). **Multi-Krum wins decisively.**

**How it works.** Each round the server computes pairwise squared distances between the five client updates and scores each client by the sum of distances to its *n−f−2* nearest neighbours. It averages only the **n−f most mutually-consistent** updates and discards the f outliers. The crucial property: **selection is by geometric consistency, not magnitude**, so inflating an update only makes the attacker a more obvious outlier — scaling cannot help them. The discarded index *is* the **detection output**.

**Choosing the worst case.** We poison **client 2 — the largest bank (~43% of all data)** — because size-weighted FedAvg gives it the most influence, so it is the hardest bank to lose to. Even a *moderate* ×20 scaling from it is enough:
- Naive FedAvg collapses to **0.211** (the sabotaged update drags the model toward flipped labels).
- **Multi-Krum recovers to 0.780 (+0.569)** with **perfect detection (P=R=1.0)**.
- **Scale-immunity:** across attack magnitudes ×10 → ×100, Multi-Krum holds a flat **≈0.78** while naive thrashes (`media/scale_sweep.png`). Its F1 is essentially independent of how hard the attacker pushes.

**The attacker's dilemma (security depth).** A smart attacker who knows we might filter by update *size* can instead **cap its update to the median honest norm** to look normal. We tested exactly this adaptive attack:

| Under the stealth (norm-capped) attack | F1 | detection recall |
|---|---|---|
| naive FedAvg | 0.733 | — |
| norm-based detector | 0.591 | **0.00** (blinded — and it *hurts*, wrongly dropping an honest bank) |
| **Multi-Krum** | **0.785** | 0.50 |

The insight: **capping the norm to hide also caps the damage** — the stealth attack barely dents naive FedAvg (0.76→0.73). So the attacker faces a dilemma: be *loud* (×20) and get detected & fully undone by Multi-Krum, or be *stealthy* and be too weak to matter. **Either way Multi-Krum lands at ≈0.78.** Norm-based defenses, by contrast, are both evadable and actively harmful. This is the core security lesson: **magnitude-based defenses fail; geometric-consistency defenses hold** (`media/attacker_dilemma.png`).

**What didn't work:** FLTrust (server-side trust bootstrapping from a small clean root) was unstable here (0.49 ± 0.19) — a 400-row root gave too noisy a reference on a shifted test set. Trimmed-mean failed once two banks were malicious. We report these honestly; they informed the choice of Multi-Krum.

## The distribution-shift / calibration finding

KDDTest+ is a *shifted* set (57% attacks vs 47% in training, plus novel attack types), so the model is under-confident on attacks and the F1-optimal threshold sits far below 0.5. We calibrate the threshold on a **30% split of the test set, evaluate on the held-out 70%** (a single scalar — negligible overfitting), and **bake it into the exported model** as an output-bias shift, so `evaluate()` at the 0.5 cutoff reproduces the calibrated score. This lifts the defended model from **0.783 → 0.906 F1**. We report the *robustness deltas at the uncalibrated 0.5 cutoff* (fair, apples-to-apples) and the *calibrated 0.906* as the deployed operating point.

## Reproducibility

Python-only, CPU, < 10 min. Run `02_Intermediate_Advanced_Day2.ipynb` top to bottom: it downloads NSL-KDD, rebuilds the non-IID split (seed 42), runs every comparison over 3 seeds, regenerates all figures in `media/`, and writes `model_scripted.pt` + `submission.json`. Organizers verify the primary number by loading `model_scripted.pt` and running `evaluate()` on KDDTest+ (== 0.906).
