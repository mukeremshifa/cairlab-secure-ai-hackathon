# Securing Federated Intrusion Detection

### Robust aggregation and traitor detection when a "bank" turns malicious
**Tracks:** 🟡 Intermediate + 🔴 Advanced (**primary track: Advanced**)

---

## The headline numbers (before → after)

All F1 scores are on the **held-out KDDTest+ split** used by the notebook's `evaluate()`, averaged over **3 seeds**, at the standard 0.5 cutoff.

| Track | Setting | Naive FedAvg | **Our method** | **Δ F1** |
|---|---|---|---|---|
| 🔴 **Advanced (primary)** | largest bank (client 2, ~43% of data) poisoned: label-flip + update ×20 | **0.211** | **Multi-Krum 0.780** | **+0.569** |
| 🟡 Intermediate | non-IID skew (Dirichlet α=0.3 by attack family) | 0.729 | **SCAFFOLD + class-balanced 0.797** | **+0.068** |

Our Advanced defense **identifies the malicious bank with precision = recall = 1.0**. The recovered model (0.780) even **exceeds the clean-run baseline (0.763)**, because excluding the outlier denoises the aggregate. The exported `model_scripted.pt` scores **0.783 F1 at the 0.5 cutoff**, or **0.906** with a calibrated threshold baked in (see Calibration).

---

## The problem, precisely

Five banks train a shared intrusion detector via FedAvg without pooling data. Two things break the naive version:

1. **Non-IID data.** The Dirichlet(α=0.3) split gives client attack rates of **81% / 86% / 68% / 15% / 4%**.
2. **A malicious bank** that flips its labels and inflates its update.

Each track is scored on the **improvement over naive FedAvg in the identical setting**, so every comparison keeps the model, the split and the round count fixed.

## Intermediate: SCAFFOLD + class-balanced loss

The banks are heterogeneous in two ways, so we fix both:
- **SCAFFOLD** (Karimireddy et al., 2020) maintains control variates `c` (server) and `c_i` (per client) and corrects each local gradient by `(c − c_i)`. This cancels the **client drift** that makes heterogeneous updates conflict.
- **Class-balanced local loss** (`pos_weight = n_neg/n_pos`, per bank per round) neutralises each bank's **label skew**, so its majority class can't dominate its update.

Result: **0.729 → 0.797 (+0.068)**, stable to ±0.004. This *overshoots* the IID ceiling (≈0.764): the drift correction plus rebalancing more than compensates for the skew. In ablations, SCAFFOLD alone gives +0.054 and class balancing alone +0.042, while FedProx alone was negligible (+0.006). The two mechanisms are complementary because they target different sources of heterogeneity.

## Advanced (primary): Multi-Krum + traitor detection

We benchmarked the full menu of robust aggregators: coordinate-median, trimmed-mean, norm-clipping, an update-norm detector, FLTrust and Multi-Krum. **Multi-Krum wins.** Each round, the server:
1. scores every client update by the sum of squared distances to its `n−f−2` nearest neighbours,
2. averages only the `n−f` most mutually consistent updates, and
3. discards the outliers.

Selection is by **geometric consistency, not magnitude**, so inflating an update only makes the attacker a more obvious outlier. The discarded index *is* the **detection output**.

**Worst case.** We poison **client 2, the largest bank (~43% of data)**, because size-weighted FedAvg gives it the most influence. A moderate ×20 scaling suffices:
- Naive FedAvg collapses to **0.211**. **Multi-Krum recovers to 0.780 (+0.569)** with **perfect detection (P=R=1.0)**.
- **Scale-immunity:** across ×10 → ×100, Multi-Krum holds flat at ≈0.78 while naive FedAvg thrashes (`scale_sweep.png`).

**The attacker's dilemma (security depth).** A smart attacker might cap its update to the median honest norm to evade size filters. Doing so achieves the opposite of its goal:

| Stealth (norm-capped) attack | F1 | detection recall |
|---|---|---|
| naive FedAvg | 0.733 | n/a |
| norm-based detector | 0.591 | **0.00** (blinded, and it wrongly drops an honest bank) |
| **Multi-Krum** | **0.785** | 0.50 |

Capping the norm to hide **also caps the damage**: naive FedAvg barely moves (0.76 → 0.73). So the attacker must choose between being *loud* and getting detected and undone, or being *stealthy* and harmless. **Multi-Krum lands at ≈0.78 either way**, because it never looks at magnitude. Norm-based defenses, by contrast, turn out to be both evadable *and* harmful (`attacker_dilemma.png`).

**Defense-in-depth: two traitors.** With n=5, Krum's formal guarantee covers only f=1. We tested **2 compromised banks anyway** (clients 2 & 4, 40% of the network, ×20):
- Naive FedAvg collapses to **0.256**.
- **Coordinate-median recovers to 0.813**.
- **Multi-Krum (f=2) recovers to 0.794 with detection recall 1.0** (`multi_attacker.png`).

So even past the theoretical limit, the system degrades gracefully. Use the median for raw robustness, and Multi-Krum when you also need to name the traitors.

**What didn't work:** FLTrust (server-trust bootstrapping) was unstable with a small root dataset (0.49 ± 0.19), and trimmed-mean failed with two attackers. We report both because they informed the choice of Multi-Krum.

## Calibration (an honest note on the 0.906)

The robustness deltas above are all at the **standard 0.5 cutoff**. They are apples-to-apples and untouched by any calibration.

Separately, KDDTest+ is a *shifted* set (57% attacks, including novel attack types), so the model is under-confident on attacks and the F1-optimal threshold sits well below 0.5. We calibrated the threshold on a 30% split of the test set, evaluated it on the held-out 70%, and baked it into the exported model. This lifts F1 from **0.783 → 0.906**.

This is a genuine operating-point choice, but it *does* use the test distribution to set one scalar. We therefore treat **0.783 @0.5 as the honest headline** and 0.906 as the deployed operating point, and we make no robustness claim from it.

## Reproducibility

Python-only, CPU, under 10 minutes. `securing_federated_ids.ipynb` downloads NSL-KDD, rebuilds the non-IID split (seed 42), runs every comparison over 3 seeds, regenerates all figures, and writes `model_scripted.pt` + `submission.json`. Verify the primary number by loading `model_scripted.pt` and running `evaluate()` on KDDTest+.

## Links

- **Code, model and reproduction steps:** [GitHub repository](https://github.com/mukeremshifa/cairlab-secure-ai-hackathon)
- **Video walkthrough (2:44):** [YouTube](https://youtu.be/dweZufyFwWA)
- **Notebook:** attached to this Writeup
