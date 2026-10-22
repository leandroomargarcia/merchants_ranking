# Use of AI

Leandro Garcia framed, guided, and checked the analysis from start to finish. He used Claude Code as a working assistant: he decided what to try, reviewed every result, asked for corrections, and made the final call. The AI wrote the code, ran the tests, and drafted what he asked for.

## How the work was done

Every stage followed the same loop:

1. Leandro stated the question or the hypothesis.
2. The AI ran it on the data and returned the numbers.
3. Leandro reviewed the result, challenged it, and asked for changes or extra checks.
4. Leandro decided, and the decision was recorded with its evidence in [`decisions.md`](decisions.md).

## By stage

| Stage | How Leandro guided and decided it | What the AI ran |
|---|---|---|
| Framing the model | Asked for alternatives to a classifier and asked to include an anomaly-detector test in the roadmap. Set each version of the target. | Compared the options and built the datasets. |
| Target | Asked to look at each suspension day by day, questioned which row stood for each positive, and decided which cases stayed out of the analysis. Asked the company which day each row summarizes. | Produced the per-merchant charts and built the dataset. |
| Variables and model | Asked to train several designs, decided which variables each model used, and what the rule captured compared with the logistic model. Chose the simpler model. | Wrote the notebooks, the repeated validation, and the metrics. |
| Score and join | Asked to join the score onto the transactions and to check coverage. | Implemented the join and the checks. |
| Rules | Set the business assumptions (margin and hold), chose the cuts, and asked to add the CNP and high-amount rules. | Computed the KPIs of each alternative. |
| Documentation and presentation | Set the structure, the style, and that each decision should show what it rests on. | Drafted `decisions.md`, the README, and the HTML. |

## Checking results

Leandro reviewed the results and asked for a correction whenever something did not hold. Among others:

- Model metrics were recomputed after finding that row order favored the positives.
- The rule for which row represents each positive was adjusted after the company said which day each row summarizes.
- Notebooks were consolidated and inconsistencies in `decisions.md` were fixed.

## What was not delegated

The business judgment, the assumptions, the choice among alternatives, and the reading of each result are Leandro's.
