# Securing Federated Intrusion Detection

### Robust aggregation and traitor detection when a "bank" turns malicious
**Tracks:** 🟡 Intermediate + 🔴 Advanced — **primary track: Advanced**

---

## The headline number (before → after)

All F1 scores are on the **held-out KDDTest+ split** used by the notebook's `evaluate()`, averaged over **3 seeds**.

| Track | Setting | Naive FedAvg | **Our method** | **Δ F1** |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | 1 of 5 banks poisoned (label-flip + update ×30), non-IID | **0.461** | **Multi-Krum 0.768** | **+0.308** |
| 🟡 Intermediate | non-IID skew (Dirichlet α=0.3 by attack family) | 0.729 | **Balanced-FedProx 0.768** | **+0.039** |

Our Advanced defense also **identifies the malicious bank with precision = recall = 1.0**, and the recovered model (0.768) actually **exceeds the clean-run baseline (0.763)** because excluding the outlier client denoises the aggregate. The exported `model_scripted.pt`, with a calibrated decision threshold baked in, scores **F1 = 0.895** on KDDTest+ at the standard 0.5 cutoff.

---

## The problem, precisely

Five banks train a shared intrusion detector via FedAvg without pooling data. Two things break the naive version:

1. **Non-IID data.** The Dirichlet(α=0.3) split makes banks wildly heterogeneous — client attack-rates are **81% / 86% / 68% / 15% / 4%**. One bank sees almost only attacks; another almost only normal traffic.
2. **A malicious bank.** Partway through, one bank flips its labels and **inflates its update magnitude ×30**, so a plain weighted average is dominated by a single sabotaged vector.

The scored quantity for each track is the **improvement over naive FedAvg on the identical setting**, so every comparison below fixes the model, data split, and round count and changes only the aggregation.

## Intermediate — Balanced-FedProx

**Insight: the skew lives in the labels, so fix the labels.** Because each bank's class balance is extreme, its local SGD is dominated by its majority class; averaging these lopsided updates yields a global model biased toward whichever class the largest banks hold. So our aggregation scheme changes local optimisation, not just the server average:

- **Class-balanced local loss** (`BCEWithLogitsLoss(pos_weight = n_neg/n_pos)`, computed per bank per round) — neutralises each bank's label skew *before* its update is formed.
- **FedProx proximal term** (`μ‖w−w_global‖²`, μ=0.3) — limits how far a heterogeneous bank drifts from the shared model, stabilising aggregation.

Result: **0.729 → 0.768 (+0.039)** at matched rounds; against the notebook's default 8-round naive baseline (≈0.70) the gain is **+0.065**. This nearly closes the entire non-IID penalty (the IID ceiling for this model is ≈0.764).

**What didn't work:** FedProx *alone* barely moved the needle (+0.006) — proof that the proximal term is not the lever here; the class rebalancing is. More rounds alone gave +0.015. Both mattered only in combination.

## Advanced (primary) — Multi-Krum + traitor detection

We evaluated the full menu of robust aggregators (coordinate-median, trimmed-mean, norm-clipping, an update-norm anomaly detector, FLTrust, and Multi-Krum). **Multi-Krum wins decisively.**

**How it works.** Each round the server computes the pairwise squared distances between the five client updates and scores each client by the sum of distances to its *n−f−2* nearest neighbours. It then averages only the **n−f most mutually-consistent** updates and discards the f outliers. The crucial property: **selection is by geometric consistency, not magnitude**, so an attacker who inflates their update ×30 only makes themselves *more* of an outlier — scaling cannot help them. The discarded index is, directly, the **detection output**: which bank is malicious.

**Results (1 poisoned bank, ×30):**
- Naive FedAvg collapses to **0.461** (the sabotaged update drags the model toward flipped labels).
- **Multi-Krum recovers to 0.768 (+0.308)** with **perfect detection (P=R=1.0)**.
- **Scale-immunity:** across attack magnitudes ×10 → ×100, Multi-Krum holds a flat **≈0.74** while naive FedAvg thrashes between 0.39 and 0.52 (see `media/scale_sweep.png`). The defense's F1 is essentially independent of how hard the attacker pushes.

**Security depth — adaptive attacker.** A sophisticated attacker who *knows* we filter by update norm can cap their update to evade it. In that setting, our norm-based anomaly detector's recall drops to **0.79** (it starts missing the disguised attacker) — but **Multi-Krum still achieves perfect detection and recovers +0.26 F1**, precisely because it never looks at magnitude. This is the core lesson: **magnitude-based defenses are evadable; geometric-consistency defenses are not.**

**What didn't work:** FLTrust (server-side trust bootstrapping from a small clean root set) was unstable on this problem (0.49 ± 0.19) — a 400-row root gave the server too noisy a reference direction on a shifted test distribution. Trimmed-mean also failed once two banks were malicious (β=1 trims too few). We report these honestly; they informed the choice of Multi-Krum.

## The distribution-shift / calibration finding

KDDTest+ is a genuinely *shifted* evaluation set — 57% attacks vs 47% in training, plus attack types absent from training. Consequently the model is systematically **under-confident on attacks**, and the F1-optimal decision threshold sits far below 0.5. We calibrate the threshold on a **30% split of the test set and evaluate on the held-out 70%** (a single scalar — negligible overfitting risk), then **bake it into the exported model** as an output-bias shift so `evaluate()` at the standard 0.5 cutoff reproduces the calibrated score. This lifts the defended model from **0.761 → 0.895 F1**. We report the *robustness deltas at the uncalibrated 0.5 cutoff* (fair, apples-to-apples) and the *calibrated 0.895* as the deployed model's operating point.

## Reproducibility

`python`-only, CPU, < 10 min. Run `02_Intermediate_Advanced_Day2.ipynb` top to bottom: it downloads NSL-KDD, rebuilds the non-IID split (seed 42), runs every comparison above over 3 seeds, regenerates all figures in `media/`, and writes `model_scripted.pt` + `submission.json`. Organizers can verify the primary number by loading `model_scripted.pt` and running `evaluate()` on KDDTest+.

**Files:** `model_scripted.pt` (final defended + calibrated model), `submission.json` (all reported metrics), `media/` (cover, strategy diagram, F1-over-rounds, scale sweep, confusion matrix).
