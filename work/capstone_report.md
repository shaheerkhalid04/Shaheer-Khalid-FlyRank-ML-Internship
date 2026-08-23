# Capstone Report - Refresh / Content Opportunity Scoring

- **Author:** Shaheer Khalid
- **Lane:** Lane 2, Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/shaheerkhalid04/Shaheer-Khalid-FlyRank-ML-Internship
- **Paper:** https://shaheerkhalid04.github.io/Shaheer-Khalid-FlyRank-ML-Internship/
- **Date:** August 2026

## 0. Abstract

A small content team can review perhaps fifty pages a week, so the question is not which pages
are declining but which to open first. I built a page-level ranking task on 1.3 million
page-months of pseudonymised daily search performance, predicting whether a page's daily search
demand falls more than 20% in the month after a cutoff date, with features drawn strictly from
the months before it. A transparent rule and two learned models were compared on identical
splits: five time-ordered cutoffs, grouped cross-validation by client, and a final sealed month
never touched during development. On that sealed month the gradient-boosted model was
indistinguishable from a two-line rule on ranking quality (AUC 0.593 against 0.591) and far worse
on the metric an editor would actually feel, capturing 0.09% of at-risk impressions in its top
200 against the rule's 4.27%, because it filled the queue with pages averaging 186 monthly
impressions instead of 28,658. The honest recommendation is to ship the rule, keep the model as a
monitoring signal only, and treat the result as decision support for editor triage rather than
evidence about how search engines behave.

## 1. Problem framing

**Decision supported:** which page a content editor opens first, given a fixed number of review
hours per week.

- **Unit of analysis:** one page (`content_hash_id`) observed at one cutoff date.
- **Output:** a ranked queue, each row carrying a score, one reason code, and an action label
  (`REFRESH`, `REVIEW`, `MONITOR`, `PROTECT`).
- **Action a human takes:** works down the list, refreshing, expanding, protecting, or
  dismissing each page.
- **Cost of a wrong call:** a false positive burns an editor hour on a page that was fine and may
  disturb a page that needed nothing. A false negative leaves a page with real audience demand
  bleeding traffic unwatched. Capacity is the binding constraint, so the expensive error is at the
  top of the list, which is why every metric is reported at a queue depth K.

**Why data helps at all:** roughly 65% of pages with real demand were losing ground in the test
month, so a yes/no flag lights up two thirds of the library and answers nothing. Ordering is the
only useful output, and the signals that determine order (demand, momentum, click-through against
ranking peers, visibility consistency) pull in different directions on different pages.

## 2. Data safety

**Used:** `fact_content_daily_performance` (every modelled number, because it carries a
`report_date` and I can prove when each value was true) and `dim_clients` (history coverage, to
size the panel limitation).

**Deliberately excluded:**

| Excluded | Why |
|---|---|
| every date column in `dim_content` | `last_optimized_date` has 45,396 non-null values and the earliest is 2026-04-24, after every cutoff I model. It is a current-state dimension with no history. `content_updated_date` is future-dated on 382,739 rows |
| `fact_content_query_90d` | its fixed 90-day window overlaps the label period, so its impression columns contain part of the answer |
| product flags and existing system scores | learning them teaches the old rule, not the world |
| `trend_direction`, `trend_pct` (starter file) | the starter label is derived from them, so both are label-derived by construction |

**Leakage risks considered:** label-derived features, overlapping feature and label windows,
decision-derived product flags, and population selection that consults the outcome window. All
four are audited in the capstone notebook with executed probes.

**Pseudonymous IDs** are used for grouping, joining and splitting only, never as features.

**Public-safety confirmation:** no client names, domains, URLs, raw queries, tokens or credentials
appear anywhere in `work/`, `docs/`, or any commit. Clients and pages exist only as opaque hashes.

## 3. Baseline

A transparent rule, frozen before modelling began:

```text
peer_ctr    = aggregate click-through of all pages in this page's position band
ctr_deficit = (peer_ctr - page ctr) / peer_ctr,  clipped to [0, 1]
score       = ctr_deficit  x  log(1 + monthly impressions)
```

Two inputs, no fitted weights, readable by a non-engineer. It is a fair comparison because it is
scored on exactly the same rows, the same label and the same splits as every model, computed in
the same notebook run.

The logarithm is a deliberate design response to evidence, not decoration. Raw demand is
negatively associated with decline in this panel (the largest pages are the most stable), so
multiplying by raw impressions turns the score into a volume ranking and degrades precision from
0.650 to 0.545 at K=200.

**Baseline on the sealed month:** AUC 0.591, P@50 0.780, P@200 0.830, impression recall@200 4.27%.

## 4. Model / analysis

**Method:** gradient boosting (`HistGradientBoostingClassifier`) with logistic regression as a
readable reference. The question is "which first?", so both are used as *rankers*: the predicted
probability is a sort key and quality is judged at a queue depth, never by accuracy.

**Target, in one sentence:** for a page with at least 50 impressions in the cutoff month, whether
its mean daily impressions in the following month fall more than 20% below its mean daily
impressions in the cutoff month.

Impressions are normalised per day because calendar months differ in length and an unnormalised
ratio partly measures February being three days short. The normalised and raw labels agree on
97.2% of rows.

**Features (11, all from the cutoff month or earlier):** `log_imp_m0`, `imp_90d`, `momentum`,
`momentum_prior`, `ctr`, `position`, `position_coverage`, `position_delta`, `ctr_deficit`,
`active_day_ratio`, `clk_m0`.

**Left out on purpose:** everything in section 2's exclusion table, plus raw `gsc_sum_position`
(kept only as the input to a weighted average) and both ID columns.

## 5. Evaluation

**Splits, three of them, because each catches something different:**

- **Grouped 5-fold by `client_hash_id`.** Pages from one client share a template, an author and a
  site, so a random split lets the model recognise the client rather than learn the pattern.
- **Time-ordered.** Train on cutoffs 2026-01 to 2026-03, validate on 2026-04.
- **Sealed month.** The 2026-05 cutoff, whose label month is June 2026, scored once at the end.

**Grouped cross-validation, training cutoffs (base rate 0.350, n = 284,096):**

| Method | AUC | P@200 | Lift |
|---|---|---|---|
| Gradient boosting | 0.649 | 0.700 | 2.00x |
| Transparent rule | 0.578 | 0.625 | 1.78x |
| Logistic regression | 0.570 | 0.535 | 1.53x |

**Sealed test, June 2026 (base rate 0.649, n = 133,719):**

| Method | AUC | P@50 | P@200 | Impression recall@200 | Median page size |
|---|---|---|---|---|---|
| Transparent rule | 0.591 | 0.780 | **0.830** | **4.27%** | **28,658** |
| Gradient boosting | 0.593 | **0.840** | 0.805 | 0.09% | 186 |
| Model x log(demand) | 0.582 | 0.780 | 0.825 | 2.66% | 14,942 |
| Logistic regression | 0.595 | 0.560 | 0.670 | 5.70% | n/a |

**The two results disagree, and that is the finding.** Within the training period the model wins
clearly. On a sealed future month all three AUCs land within 0.004 of each other, and the model
loses decisively on protected traffic.

**Error analysis:**

- The model is badly calibrated under drift: lowest decile predicted 0.146 against an observed
  0.402, highest predicted 0.639 against 0.733. It was trained where 35% of pages declined and
  tested where 65% did. The ranking survives this; the probabilities should not be shown to anyone.
- 117 rows are confident (p > 0.8) and wrong. The largest grew 108x month-over-month and then
  held: a spike the model read as fragility.
- Permutation importance puts `active_day_ratio` first (0.049), then `ctr` (0.032) and `momentum`
  (0.028). `position` is slightly negative, meaning shuffling it did not hurt.

**Leakage probes:**

- Deliberate leak: adding next-month impressions drove AUC to 0.9997 against an honest 0.5925.
- Memorisation gap: random 5-fold 0.7202 against grouped 5-fold 0.6488, a 7-point gap that is
  exactly why no random split appears in any reported number.

## 6. Interpretation

The model's leading signal is one the rule does not use: how many days of the month a page was
visible at all. I tried to fold that discovery back into the rule and **failed**. Decline rate
across active-day bands runs 20.3%, 44.8%, 30.2%, 39.1%, which is not monotone, so there is no
threshold to write. Whatever the model uses is an interaction, not a readable cut point, and the
frozen rule therefore stands unchanged.

Two negative results worth stating plainly:

1. **The model does not beat the rule out of period.** It optimised the metric it was given, and
   that metric counted pages as if they were equal. A queue of 200 near-invisible pages scores
   well on precision and protects almost no traffic.
2. **The base rate is not stable.** It runs 0.21, 0.28, 0.50, 0.55, 0.65 across the five cutoffs
   as the panel shifts from rapid growth into contraction. The thing being predicted genuinely
   changes over eight months, which is arguably the most actionable signal in the whole project
   and one no per-page queue will ever surface.

A surprise from earlier weeks that shaped everything: `gsc_avg_position = 0` means "no position
recorded", not rank zero. 163,189 rows in a single month carry it while still reporting
impressions. Averaging position naively put pages at an impossible "position 0.12" at the top of
my first queue.

## 7. Recommendation

1. **Ship the rule, not the model.** Same ranking quality on unseen future data, 47x more
   protected traffic, readable in one line, no training pipeline, cannot drift silently.
2. **Change the objective before training anything else.** Select future models on
   impression-weighted recall at K, so being right about an invisible page counts for what it is
   worth.
3. **Surface position coverage in the queue.** Pages whose impressions mostly carry no ranking
   position decline at 67.8% against 47.1%. They belong in the queue (filtering them lowers
   precision at every K I tested) but the reason code should say the position data is thin, not
   that the title underperforms.
4. **Split off the six-figure-impression, single-digit-click pages.** They dominate the top of the
   rule's queue and are rarely a title problem. They need a query-mix diagnosis, not a rewrite.
5. **Recalibrate monthly and treat the base rate as a portfolio health metric.**

**How an editor uses this tomorrow:** open `work/outputs/capstone_action_queue.csv`, work down
from rank 1, and read the reason code before acting. The file deliberately carries no label
column: a queue must not hand its reader the answer.

**Confidence and limits:** this is decision support, offered as observed and directional. It ranks
pages for human attention. It does not promise that editing a page recovers its traffic, and it
says nothing about how search engines work.

## 8. Reproducibility

From a clean clone:

```bash
python -m venv .venv && .venv/Scripts/pip install -r requirements.txt
# request access to the gated dataset once, then authenticate:
.venv/Scripts/hf auth login
.venv/Scripts/jupyter nbconvert --to notebook --execute --inplace work/notebooks/capstone.ipynb
```

- **Seeds:** `SEED = 42` throughout (model init, cross-validation shuffling, permutation
  importance, sampling).
- **Row order is sorted explicitly** after every warehouse pull. DuckDB scans partitions in
  parallel and does not guarantee ordering, and two identical pulls otherwise produced two
  different AUCs. This is the single most important reproducibility detail in the project.
- **Environment:** Python 3.12.6, duckdb 1.5.5, pandas 3.0.3, scikit-learn 1.9.0,
  matplotlib 3.11.1, huggingface_hub 1.28.0.
- **Cost:** the warehouse scan is roughly five minutes and caches to `work/outputs/`. Every later
  step reads the cache, so re-runs are fast and do not hit rate limits.

**Sealed holdout receipts, both committed:**

- The frame builder: the cutoff-construction cell in
  [`work/notebooks/capstone.ipynb`](notebooks/capstone.ipynb), which defines all five cutoffs
  including the sealed 2026-05 row set.
- The metrics it produced: [`work/outputs/capstone_metrics.json`](outputs/capstone_metrics.json).

**Population disclosure:** eligibility is "at least 50 impressions in the cutoff month", which
uses no column from the outcome window. No survivorship filter was applied, and clients absent
from a month are simply absent rather than dropped by a rule.

## 9. Acknowledgments and data credit

Built on the FlyRank ML Internship dataset, https://flyrank.ai. The warehouse release is real,
pseudonymised search performance data, and the fact that it is real is what made the negative
result in this report worth reporting at all. Thanks to the FlyRank team for opening it up, and
for a curriculum that put leakage hunting and honest validation ahead of scores.
