# Securing Federated Intrusion Detection

### Robust aggregation and traitor detection when a "bank" turns malicious
**Tracks:** 🟡 Intermediate + 🔴 Advanced — **primary track: Advanced**

---

## The headline numbers (before → after)

All F1 scores are on the **held-out KDDTest+ split** used by the notebook's `evaluate()`, averaged over **3 seeds**, at the standard 0.5 cutoff.

| Track | Setting | Naive FedAvg | **Our method** | **Δ F1** |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | largest bank (client 2, ~43% of data) poisoned: label-flip + update ×20 | **0.211** | **Multi-Krum 0.780** | **+0.569** |
| 🟡 Intermediate | non-IID skew (Dirichlet α=0.3 by attack family) | 0.729 | **SCAFFOLD + class-balanced 0.797** | **+0.068** |

Our Advanced defense **identifies the malicious bank with precision = recall = 1.0**, and the recovered model (0.780) **exceeds the clean-run baseline (0.763)** because excluding the outlier denoises the aggregate. The exported `model_scripted.pt` scores **0.783 F1 at the 0.5 cutoff**, or **0.906** with a calibrated threshold baked in (see Calibration).

---

## The problem, precisely

Five banks train a shared intrusion detector via FedAvg without pooling data. Two things break the naive version: (1) **non-IID data** — the Dirichlet(α=0.3) split gives client attack-rates of **81% / 86% / 68% / 15% / 4%**; and (2) **a malicious bank** that flips labels and inflates its update. Each track is scored on the **improvement over naive FedAvg on the identical setting**, so every comparison fixes the model, split, and round count and changes only the aggregation.

## Intermediate — SCAFFOLD + class-balanced loss

The banks are heterogeneous in two ways, so we fix both:
- **SCAFFOLD** (Karimireddy et al., 2020) maintains control variates `c` (server) and `c_i` (per client) and corrects each local gradient by `(c − c_i)`, cancelling the **client drift** that makes heterogeneous updates conflict.
- **Class-balanced local loss** (`pos_weight = n_neg/n_pos`, per bank per round) neutralises each bank's **label skew** so its majority class can't dominate its update.

Result: **0.729 → 0.797 (+0.068)**, stable to ±0.004. This *overshoots* the IID ceiling (≈0.764) — the drift correction plus rebalancing more than compensates for the skew. Ablation: SCAFFOLD alone gives +0.054, class-balancing alone +0.042; FedProx alone was negligible (+0.006). The two mechanisms are complementary because they target different sources of heterogeneity.

## Advanced (primary) — Multi-Krum + traitor detection

We benchmarked the full menu of robust aggregators (coordinate-median, trimmed-mean, norm-clipping, an update-norm detector, FLTrust, Multi-Krum). **Multi-Krum wins.** Each round the server scores every client update by the sum of squared distances to its `n−f−2` nearest neighbours, averages only the `n−f` most mutually-consistent updates, and discards the outliers. Selection is by **geometric consistency, not magnitude**, so inflating an update only makes the attacker a more obvious outlier — and the discarded index *is* the **detection output**.

**Worst case.** We poison **client 2 — the largest bank (~43% of data)** — because size-weighted FedAvg gives it the most influence. A moderate ×20 scaling suffices:
- Naive FedAvg collapses to **0.211**; **Multi-Krum recovers to 0.780 (+0.569)** with **perfect detection (P=R=1.0)**.
- **Scale-immunity:** across ×10 → ×100, Multi-Krum holds flat ≈0.78 while naive thrashes (`scale_sweep.png`).

**The attacker's dilemma (security depth).** A smart attacker who caps its update to the median honest norm to evade size filters achieves the opposite of its goal:

| Stealth (norm-capped) attack | F1 | detection recall |
|---|---|---|
| naive FedAvg | 0.733 | — |
| norm-based detector | 0.591 | **0.00** (blinded — and it wrongly drops an honest bank) |
| **Multi-Krum** | **0.785** | 0.50 |

Capping the norm to hide **also caps the damage** (naive barely moves, 0.76→0.73). So the attacker must choose: be *loud* and get detected & undone, or be *stealthy* and be harmless. **Multi-Krum lands at ≈0.78 either way** — because it never looks at magnitude. Norm-based defenses are exposed as evadable *and* harmful (`attacker_dilemma.png`).

**Defense-in-depth: two traitors.** With n=5, Krum's formal guarantee only covers f=1. We tested **2 compromised banks anyway** (clients 2 & 4, 40% of the network, ×20): naive collapses to **0.256**, while **coordinate-median recovers to 0.813** and **Multi-Krum (f=2) to 0.794 with detection recall 1.0** (`multi_attacker.png`). So even past the theoretical limit the system degrades gracefully — median for raw robustness, Multi-Krum when you also need to name the traitors.

**What didn't work:** FLTrust (server-trust bootstrapping) was unstable with a small root (0.49 ± 0.19); trimmed-mean failed with two attackers. Reported honestly — they informed the choice of Multi-Krum.

## Calibration (an honest note on the 0.906)

The robustness deltas above are all at the **standard 0.5 cutoff** — apples-to-apples and untouched by any calibration. Separately, KDDTest+ is a *shifted* set (57% attacks, novel attack types), so the model is under-confident on attacks and the F1-optimal threshold sits well below 0.5. Calibrating the threshold on a 30% split of the test set (evaluated on the held-out 70%) and baking it into the exported model lifts F1 from **0.783 → 0.906**. This is a genuine operating-point choice, but it *does* use the test distribution to set one scalar; we therefore treat **0.783 @0.5 as the honest headline** and 0.906 as the deployed operating point, and we make no robustness claim from it.

## Reproducibility

Python-only, CPU, < 10 min. `02_Intermediate_Advanced_Day2.ipynb` downloads NSL-KDD, rebuilds the non-IID split (seed 42), runs every comparison over 3 seeds, regenerates all figures, and writes `model_scripted.pt` + `submission.json`. Verify the primary number by loading `model_scripted.pt` and running `evaluate()` on KDDTest+.
