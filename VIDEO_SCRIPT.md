# 3-Minute Video Script — Securing Federated Intrusion Detection

Target: ≤ 3:00. Record screen with `media/` figures + the notebook. Times are cumulative.

---

**[0:00–0:25] Hook & setup**
> "Five banks want one fraud/intrusion detector but can't pool data, so they use federated
> learning — each trains locally, the server averages the updates. Two things break that: the
> banks see very different traffic, and one bank might be malicious. We fixed both; the
> malicious-bank case is our primary track."

*(Show cover.png.)*

**[0:25–1:05] The non-IID fix (Intermediate)**
> "Our five banks are extremely skewed — attack-rates from 4% to 86%. Plain FedAvg drops to
> 0.73 F1. The banks differ in two ways, so we fix both: SCAFFOLD control variates cancel the
> drift between them, and a class-balanced loss neutralises the label skew. That reaches 0.80 —
> actually beating the IID ceiling."

*(Show intermediate_f1_rounds.png.)*

**[1:05–2:05] The attack and the defense (Advanced — primary)**
> "Now the *largest* bank — client 2, holding 43% of the data — turns malicious. That's the
> worst case, because size-weighted FedAvg trusts it most. It flips its labels and scales its
> update just twenty-fold, and naive FedAvg collapses from 0.76 to 0.21."

*(Show advanced_f1_rounds.png — the red 'attacked' line.)*

> "Our defense is Multi-Krum: the server keeps only the updates that are geometrically
> consistent with each other and discards the outliers. Because it selects by *distance, not
> magnitude*, scaling just makes the attacker a more obvious outlier. We recover to 0.78 — a
> +0.57 F1 improvement — and identify the malicious bank with perfect precision and recall."

*(Show strategy_diagram.png, then scale_sweep.png.)*

> "This chart is the punchline: from ×10 to ×100, naive FedAvg thrashes but Multi-Krum stays
> flat. It's immune to attack magnitude."

**[2:05–2:40] The attacker's dilemma (what sets it apart)**
> "So a smart attacker hides instead — it caps its update to look normal-sized and evade any
> size-based filter. We tested that. Norm-based detection is blinded — recall drops to zero,
> and it even hurts accuracy by dropping an honest bank. But here's the catch: capping the
> update to hide it also strips its power — naive barely moves, 0.76 to 0.73. So the attacker
> is stuck: be loud and get caught, or be quiet and be harmless. Multi-Krum wins either way,
> because it never looks at magnitude. That's the real security lesson."

*(Show attacker_dilemma.png.)*

**[2:40–3:00] Close**
> "Everything's reproducible in one notebook, under ten minutes on a CPU, over multiple seeds.
> Final model: 0.91 F1 on the held-out test set, robust to a malicious bank, and it tells you
> which bank to kick out. Thanks for watching."

*(Show confusion_matrix.png, then the repo/README.)*

---

**Recording tips:** speak to the figures, not the code; one figure per beat; end on the cover
or README so the repo link is visible.
