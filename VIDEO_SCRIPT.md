# 3-Minute Video Script — Securing Federated Intrusion Detection

Target: ≤ 3:00. Record screen with `media/` figures + the notebook. Times are cumulative.

---

**[0:00–0:25] Hook & setup**
> "Five banks want one fraud/intrusion detector but can't pool data, so they use federated
> learning — each trains locally, the server averages the updates. Two things break that:
> the banks see very different traffic, and one bank might be malicious. We fixed both, and
> the malicious-bank case is our primary track."

*(Show cover.png.)*

**[0:25–1:05] The non-IID fix (Intermediate)**
> "Our five banks are extremely skewed — attack-rates from 4% to 86%. Plain FedAvg drops to
> 0.73 F1. The key insight: the skew is in the *labels*, so we rebalance each bank's loss and
> add a FedProx proximal term to stop drift. That recovers to 0.77 — closing almost the whole
> non-IID gap."

*(Show intermediate_f1_rounds.png.)*

**[1:05–2:10] The attack and the defense (Advanced — primary)**
> "Now one bank turns malicious: it flips its labels and inflates its update thirty-fold.
> Naive FedAvg collapses from 0.76 to 0.46 — the sabotaged update dominates the average."

*(Show advanced_f1_rounds.png — the red 'attacked' line.)*

> "Our defense is Multi-Krum: the server keeps only the updates that are geometrically
> consistent with each other and discards the outliers. Because it selects by *distance, not
> magnitude*, the attacker's ×30 scaling just makes them a more obvious outlier. We recover to
> 0.77 — a +0.31 F1 improvement — and we identify the malicious bank with perfect precision
> and recall."

*(Show strategy_diagram.png, then scale_sweep.png.)*

> "This chart is the punchline: as the attack gets stronger — ×10 to ×100 — naive FedAvg
> thrashes, but Multi-Krum stays flat. It's immune to attack magnitude."

**[2:10–2:40] Security depth (what sets it apart)**
> "We stress-tested an *adaptive* attacker who knows we filter by update size and hides under
> the radar. Norm-based defenses start missing it — detection recall falls to 0.79 — but
> Multi-Krum still catches it perfectly, because it never looks at magnitude. That's the real
> lesson: magnitude defenses are evadable, geometric ones aren't. We also tried FLTrust; it
> was unstable here, and we report that honestly."

**[2:40–3:00] Close**
> "Everything's reproducible in one notebook, under ten minutes on a CPU, averaged over
> multiple seeds. Final model: 0.90 F1 on the held-out test set, robust to a malicious bank,
> and it tells you which bank to kick out. Thanks for watching."

*(Show confusion_matrix.png, then the repo/README.)*

---

**Recording tips:** speak to the figures, not the code; keep one figure on screen per beat;
end on the cover or README so the link is visible.
