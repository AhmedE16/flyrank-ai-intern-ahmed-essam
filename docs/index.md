# CTR Opportunity Scoring: Finding Pages That Under-Capture Clicks Relative to Their Search Position

## Abstract
This project asks which pages a content reviewer should check first when search
visibility isn't converting into clicks. Using FlyRank's anonymized search
warehouse (March 2026, ~101,000 pages across 44 clients), I built a transparent
baseline rule comparing each page's CTR to its position tier's expected CTR, then
trained a Random Forest on two non-circular features (average position, session
count) validated with a client-grouped split. The model reached Precision@20 =
0.50 versus the baseline's 0.05 — a 10x improvement — while an earlier version
using click/impression-derived features scored a suspicious 1.00, which I
identified as label leakage and corrected.

## Introduction
Reviewers at a content team have limited time and can't manually audit every
page's search performance. Today, without a systematic tool, they either review
in arbitrary order or use a hand-written rule like "flag anything below average
CTR." This project supports one decision: which pages should a reviewer open
first to check for a title/meta/snippet problem? The action that follows is
concrete — rewrite a title, adjust a meta description, or confirm the page is
fine as-is. Getting this wrong costs real time: a false positive wastes a
reviewer's limited attention, while a false negative lets a genuine
under-performing page go unreviewed indefinitely.

## Data
Source: FlyRank internship warehouse release, table `fact_content_daily_performance`,
filtered to `month = '2026-03'` — a mid-panel month, deliberately not the final
sealed month reserved for later evaluation. One row = one page, aggregated to
the month level (content_hash_id x client_hash_id). After filtering to pages
with at least 100 monthly impressions (so CTR is a stable signal rather than
noise from near-zero traffic), the working set is 101,441 pages across 44
clients. I excluded the query-level table (`fact_content_query_90d`) this
round, since mixing grains risked leakage through aggregates that already
encode outcome information. No client names, URLs, or raw queries appear
anywhere in this analysis — all identifiers are pseudonymized hashes.

## Methodology
**Label:** a page is "low CTR" if its CTR falls below the 25th percentile CTR
for its own position tier (1-3, 4-10, 11-20, 21+). This is a same-window
definition, not a future outcome — a known limitation, addressed below.

**Baseline:** a rule scoring pages by the gap between their tier's mean CTR
(computed from the training clients only) and their actual CTR, filtered to
pages with real visibility and a reasonable position.

**Model:** Random Forest (200 trees, max depth 6, class-weighted), compared
against Logistic Regression and the baseline. Features: `avg_position`,
`sessions_month` only. I deliberately excluded `clicks_month` and
`impressions_month` — since CTR is mathematically `clicks / impressions`, and
the label is a threshold on CTR, including either would let the model
reconstruct the label algebraically rather than learn a real pattern. An
earlier version that included these features scored Precision@20/@50 = 1.00
with zero false positives — the unmistakable signature of leakage. Removing
them and keeping only position and sessions produced an honest, more modest
result.

**Validation:** client-grouped split (70/30), not a random row split — pages
from the same client can share structural similarities, so grouping ensures the
reported score reflects generalization to genuinely unseen clients.

## Results

| Method | Precision@20 | Precision@50 |
|---|---|---|
| Baseline rule | 0.05 | 0.04 |
| Logistic Regression | 0.35 | 0.28 |
| Random Forest | 0.50 | 0.40 |

Of the Random Forest's top 50 picks, 31 were false positives — a believable,
non-zero count that confirms the model is learning a real pattern rather than
reconstructing the label from leaked information. Permutation importance shows
`avg_position` dominating over `sessions_month`.

![Model vs Baseline](../work/outputs/charts/model_vs_baseline.png)
![Feature Importance](../work/outputs/charts/feature_importance.png)

## Limitations & Honest Framing
This result is observed and directional, not causal — I cannot claim that fixing
a flagged page's title/meta will recover clicks; that requires an actual
experiment. The label is a same-window proxy (this month's CTR vs. this month's
tier average), not a genuine future outcome; a stronger version would predict
next month's CTR from this month's features. The data itself is an unbalanced
panel — client tracking-history depth varies, and this is a single 31-day
window, so what looks like "low CTR" here could partly reflect normal monthly
fluctuation rather than a persistent issue. Of the Random Forest's top 50 picks,
several false positives were pages at very strong positions (near #1) that may
have naturally lower CTR due to branded/navigational search intent, not an
actual content problem.

## Ranked Recommendations
Based on the model's ranking, a reviewer's action playbook:
1. **Top-tier flags (highest score):** review title and meta description first —
   these pages combine strong position with a real CTR shortfall against peers
   in the same tier.
2. **Mid-tier flags:** treat as lower-confidence candidates; check query intent
   before assuming a content problem, since some flagged pages may serve
   navigational/branded queries where lower CTR is expected.
3. **Do not treat a high score as proof of a problem** — it's a prioritization
   signal for limited reviewer time, not a diagnosis.

## Reproducibility
Full pipeline and code: [github.com/AhmedE16/flyrank-ai-intern-ahmed-essam](https://github.com/AhmedE16/flyrank-ai-intern-ahmed-essam)
— see `work/notebooks/w02_ml_task_framing.ipynb` through `w05_model.ipynb` and
`work/notebooks/capstone.ipynb` for the complete, executed step-by-step build.

## Acknowledgments
Built on the FlyRank ML Internship dataset. Data provided by [FlyRank](https://flyrank.ai).
