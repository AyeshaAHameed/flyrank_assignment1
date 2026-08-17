# Capstone Report — Structured Content Archetype Clustering

- **Author:** Khatri (AyeshaHameed)
- **Lane:** Lane 3 — Structured Content Archetype Clustering
- **Repo:** github.com/AyeshaHameed/flyrank_assignment1
- **Date:** 2026-08-17

---

## 1. Problem framing

**Decision supported:** which broad editorial action — protect, improve, rewrite, merge, prune, or
monitor — to apply to a *group* of similarly-behaving content pages, rather than deciding
page-by-page.

**Unit of analysis:** individual content page (`content_id`), grouped into one of 6 clusters.

**Output:** a cluster assignment per page, plus a recommended action per cluster.

**Who acts on it:** a content strategist with limited review time, working across a large
inventory (30,000 pages) who needs to prioritize by group instead of reviewing every page
individually.

**Cost of a wrong call:** a mislabeled or unstable cluster misdirects review effort at scale —
e.g. flagging a healthy, well-ranking page for a rewrite it doesn't need, or leaving a genuinely
stale, poorly-ranked group unattended because it was folded into a "monitor" bucket.

**Why data/ML helps here:** with 30,000 pages, no strategist can review each one individually.
A fixed if/else rule (see Baseline, Section 3) can only encode interactions a human already
anticipated. Clustering surfaces groupings from the data itself, which can then be checked against
known-good signals rather than assumed in advance.

---

## 2. Data safety

**Data used:** `content_refresh_anonymized.csv` — anonymized FlyRank internship warehouse release,
~30,000 pages, single snapshot.

**Features used for clustering:** `word_count`, `char_count`, `ctr`, `avg_position`,
`engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `days_since_last_update`, one-hot encoded
`content_type` and `main_intent`.

**Columns deliberately excluded:**
- `trend_direction`, `trend_pct` — outcome-adjacent, label-derived fields. Confirmed as leakage
  risk in W03; excluded from every clustering input. Used only afterward, if at all, to describe
  clusters — never to form them.
- `content_id`, `client_id` — pseudonymous identifiers, used only for row identity/joins, never as
  model features.

**Confirmation:** no client names, domains, URLs, or private queries appear anywhere in `work/`.
All data is the public-safe anonymized release.

---

## 3. Baseline

**The baseline (from W04):** a transparent, weighted rule-based scorer combining two signals
validated with real bucket tables before being encoded:
- Visibility weight: 0.45
- Tier-normalized CTR gap weight: 0.40
- Staleness weight: 0.15

This produced a reason code and an action label per page. It's a fair comparison point because it
runs on the exact same underlying signals (CTR, position, staleness) that the clustering model
also uses — the difference is *how* those signals are combined (a fixed hand-written formula vs.
data-driven grouping), not *what* data each one sees.

**Baseline vs. model comparison:** *[not yet run — pending: join W04's scored output back into this
notebook on `content_id` and compare within-bucket variance to the cluster within-group variance
shown in Section 4. Fill this in before final submission — do not submit with this placeholder.]*

---

## 4. Model / analysis

**Method:** KMeans clustering, k=6, `n_init=25`, `random_state=42`. Chosen because Lane 3 is
explicitly unsupervised — there's no pre-existing archetype label to predict, and KMeans is the
simplest, most interpretable way to discover natural groupings from continuous signals.

**Why k=6:** silhouette score peaked at k=2 (0.533) but that split (2,106 vs 27,894 pages) was a
trivial, uninformative binary division — not useful for mapping to 6 distinct editorial actions.
From k=3 onward, silhouette rose gradually to a shallow peak near k=7 (0.333), with no sharp
elbow — consistent with real but soft cluster structure. k=6 was chosen to align cluster count
directly with the 6-action taxonomy (protect/improve/rewrite/merge/prune/monitor), trading a
marginal (~0.004) silhouette gain at k=7 for interpretability.

**Feature list:** `word_count`, `char_count`, `ctr`, `avg_position`, `engagement_rate`,
`scroll_rate`, `ai_traffic_pct`, `days_since_last_update`, `content_type` (one-hot),
`main_intent` (one-hot). All numeric features standardized before clustering.

**Left out on purpose:** `trend_direction`, `trend_pct` (leakage risk — see Section 2); pseudonymous
IDs (grouping only).

**Target/proxy definition, one sentence:** there is no supervised target — the cluster assignment
itself is the output, validated by internal cohesion (silhouette score) and by whether resulting
groups differ meaningfully on independently-validated signals (CTR-vs-position, staleness) rather
than by agreement with any ground-truth label.

---

## 5. Evaluation

**Split / validation design:** no train/test split in the supervised sense. Instead: (a) KMeans fit
across k=2–8, k selected via elbow + silhouette score; (b) stability check — refit at k=6 with 3
different random seeds and compare cluster assignments via Adjusted Rand Index (ARI).

**Stability results:**
- ARI seed1 vs seed2: **0.991** — near-perfect agreement
- ARI seed1 vs seed3: **0.747** — moderate agreement

**What this means:** two of three seeds landed on essentially identical clusters; the third shows
meaningful drift, indicating a subset of pages sit near cluster boundaries and their exact
membership is sensitive to initialization. This is expected with k=6 given continuous, overlapping
underlying signals — it does not invalidate the core archetypes, but individual borderline pages'
cluster membership should be treated as soft, not fixed.

**Cluster sizes (final run, n_init=25):**

| Cluster | Pages |
|---|---|
| 0 | 3,096 |
| 1 | 2,009 |
| 2 | 19,332 |
| 3 | 5,378 |
| 4 | 139 |
| 5 | 46 |

**Error analysis / notable finding:** Cluster 4 (139 pages) shows a mean CTR of **40.5**, wildly
outside the range of every other cluster (0.25–2.06). Since CTR is conventionally bounded well
under this range in this dataset, this is very likely a **data artifact** (unit or formula issue
in a small subset of rows) rather than a genuine high-performing archetype. This cluster's
recommended action is therefore "flag for data-quality review," not a confident editorial call —
see Limitations.

---

## 6. Interpretation

Cluster profiles on the two independently-validated W04 signals (CTR, avg_position) plus staleness:

| Cluster | CTR (mean) | Avg. position (mean) | Days since update (mean) | Plain-language read |
|---|---|---|---|---|
| 1 | 1.17 (best) | 5.79 (best) | 27 (freshest) | **Champions** — ranking well, clicking well, recently updated |
| 0 | 0.25 | 21.03 (worst) | 100 (stalest) | **Struggling** — poor rank, very stale content |
| 2 | 0.26 | 16.79 | 38 | **Unremarkable middle** — the largest group, no strong signal either way |
| 3 | 0.26 | 16.19 | 49.7 | Similar to cluster 2, moderately staler |
| 4 | 40.5 (anomalous) | 6.36 | 40 | Likely **data artifact** — see Section 5 |
| 5 | 2.06 | 20.0 (poor) | 41 | Small, poor-position edge group |

**What the clustering actually found:** a clear "best" archetype (cluster 1) and a clear "worst"
archetype (cluster 0) separated mainly by position and staleness — consistent with the
CTR-vs-position relationship already confirmed in W04. The majority of pages (clusters 2 and 3,
~24,700 pages combined) sit in an undifferentiated middle with no strong signal in either
direction — a genuine, honest finding, not a modeling failure: most content is neither excellent
nor urgently broken.

**Surprise / negative result:** the middle two clusters (2 and 3) are only weakly distinguished
from each other (similar CTR and position, moderate staleness gap) — the model did not find as
much fine-grained structure in the "average" majority of pages as initially hoped. This is reported
honestly rather than forced into an artificially sharper distinction.

---

## 7. Recommendation

Corrected action mapping, based on actual cluster profiles (not assumed in advance):

| Cluster | Pages | Action | Reasoning |
|---|---|---|---|
| 1 | 2,009 | **Protect** | Best position and CTR, freshest — already winning, leave as-is |
| 0 | 3,096 | **Rewrite** | Worst position, 100 days stale, weak CTR — needs substantive refresh |
| 2 | 19,332 | **Monitor** | Largest cluster, unremarkable on all signals — re-check periodically |
| 3 | 5,378 | **Improve** | Similar to cluster 2 but staler — freshness/metadata pass likely helps |
| 4 | 139 | **Monitor (flagged)** | CTR anomalous vs. all other clusters — data-quality review before acting |
| 5 | 46 | **Merge** | Small group, poor position — candidate for merging into stronger related pages |

**How a strategist uses this tomorrow:** instead of reviewing 30,000 pages individually, review by
cluster — prioritize cluster 0 (3,096 pages, clearly worst-performing) for rewrite work first,
leave cluster 1 alone, and treat cluster 2/3 (the majority, ~24,700 pages) as low-priority
monitoring rather than spending review time there.

**Confidence and limits:** confidence is high for clusters 0 and 1 (clear, well-separated
signals). Confidence is lower for clusters 2 and 3 (weak separation between them) and cluster 4
(likely data artifact — do not act on this cluster without first verifying the CTR values).

---

## 8. Limitations

- Single snapshot — no time-aware validation; archetype stability over time is unverified with this
  release.
- No ground-truth labels — validation is entirely internal (silhouette, seed stability), not
  accuracy against a known-correct grouping.
- Cluster stability is moderate, not perfect: ARI of 0.747 between two of three seeds means some
  pages near cluster boundaries have soft, not fixed, membership.
- Cluster 4's extreme CTR values are very likely a data artifact and should not be read as a real
  high-performing archetype without further data-quality investigation.
- Clusters 2 and 3 are only weakly separated — the model did not find strong fine-grained structure
  within the "average" majority of the dataset.
- All findings are **observed, directional, and decision-support** — this work does not claim
  causal effect of any recommended action, and does not model or predict Google's ranking
  algorithm.
- Findings are based on an anonymized sample; client- or topic-specific effects may not generalize.

---

## Reproducibility

- Notebook: `work/notebooks/capstone_clustering.ipynb`, runs top-to-bottom from a fresh clone.
- Random seed: `RANDOM_SEED = 42` (primary); stability checked additionally at seeds 1 and 123.
- KMeans: `n_clusters=6, n_init=25, random_state=42`.
- Environment: see `requirements.txt` in repo root.
- To reproduce: clone repo → `pip install -r requirements.txt` → open
  `work/notebooks/capstone_clustering.ipynb` → Run All.
