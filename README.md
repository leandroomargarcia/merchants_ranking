# Merchant Risk Scoring · Clip DS Challenge 2026

A merchant risk score to decide which transactions are approved and which merchants are paid immediately.

Every decision, with its evidence and limits, is in [`docs/decisions.md`](docs/decisions.md).

## The problem

Clip groups merchants into affiliations that issuers evaluate together: a few risky merchants lower approval for everyone. The challenge asks to (1) train a model, (2) score each merchant and join the score onto the transactions, and (3) propose pre- and post-authorization rules, measuring model quality and business impact separately.

## How it was solved

**Target.** `is_suspended` is a daily state, not an event. Each merchant with an observed suspension start contributes **one positive row: their first suspended day**. That row summarizes activity through the previous day (confirmed by the company), so it does not use information from after the suspension. Negatives are merchants never suspended in April–September, one active day drawn with the same date mix as the positives. That leaves 12 positives and about 33,500 negatives. Suspended merchants who show no suspicious activity before the start are excluded (likely a documentation or compliance suspension): the reported recall is over suspensions that are visible in the activity.

**Model.** L2-regularized logistic regression, 4 variables from the merchant snapshot: recent approval (the worse of 7 and 30 days), lifetime approval, a "no activity in 7 days" flag, and the worst day of declines in 30 days. Validation is by merchant: 5 stratified folds repeated with 10 seeds. Model KPIs: PR-AUC and positives captured in the top 5% of the score (10.6 of 12).

**Score and join.** Each merchant is scored on their October 1 row and the score is joined by `merchant_id` onto the 7.08 million October–December transactions (100% coverage). Those transactions were not used for training: they are the held-out set for measuring impact.

**Rules.** Cuts are score percentiles (the probability is not calibrated) and are combined with the merchant's volume:
- *Pre-authorization:* decline low volume with risk ≥ 90% and medium volume with risk ≥ 99%.
- *Post-authorization:* manual review for high volume with risk ≥ 95%, a 7-day hold for merchants with no activity, and hold the transaction if it is CNP of $500 or more, or $20,000 or more of any type.
- *Bonus:* three packages over approved transactions (low, medium, high). The main one is low.

With an assumed Clip margin of 1% and a 7-day hold, the decline avoids 12.3% of fraud dollars while declining 1.78% of good TPV. The decline plus the score hold covers 41.5%. With the CNP and amount rules, what is avoided or held reaches about 72%, and what passes with no control falls to about 28%. Detail, trade-offs, and checks are in section 5 of `decisions.md`.

## Limitations

- With 12 positives, any metric is noisy; one case moves recall by 8 points.
- The rule, the variables, and the cuts were chosen by looking at the same data. There is no further held-out period.
- The score is trained on suspension, not on fraud: it ranks the riskiest 5% well and almost nothing inside the rest (the riskiest 5% concentrates 36% of fraud).
- Clip's margin and the 7-day hold are assumptions that are not in the data.

## How to reproduce

Requires [uv](https://docs.astral.sh/uv/) and Python 3.12 or newer.

```bash
uv sync
```

The data provided by the company is not versioned. Place it in `data/`:

```text
data/merchant_rating_featureset_sample.parquet
data/txn_sample.parquet
```

Run the notebooks in order, from the `notebooks/` folder, because they resolve paths from there:

```bash
cd notebooks
uv run --with jupyter jupyter lab
```

| Notebook | What it does | Output |
|---|---|---|
| `01_EDA` | Explores the data | — |
| `02_suspensiones` | Each suspension, day by day | `docs/suspensiones*`, `docs/variables/` |
| `03_dataset` | Builds the dataset: one row per merchant | `data/processed/dataset.parquet` |
| `04_features` | Builds the features | `data/processed/features.parquet` |
| `05_logistica` | Trains and compares the rule and the logistic model; scores October | `docs/modelo/`, `data/processed/score_octubre.parquet` |
| `06_transacciones_con_score` | Joins the score onto the transactions | `data/processed/transacciones_con_score.parquet` |
| `07_reglas` | Pre- and post-authorization rules and business KPIs | `docs/reglas/` |

`requirements.txt` is an export of `uv.lock` for anyone not using uv: `pip install -r requirements.txt`.

## Layout

```text
data/            company data and derived files (not versioned)
docs/            decisions, charts, and result tables
guideline/       challenge brief
notebooks/       the pipeline, in order
presentacion/    the presentation
```
