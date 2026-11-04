# Causal Engine: case study

A configurable causal-inference engine for enterprise data. A company connects a dataset, states the business question and what its experts already know, and the engine estimates the effect of an intervention, tries to break its own estimate, and reports how far the result can be trusted. There is no LLM anywhere in the path that produces numbers.

*This repository is a case study. The source code is private and available for technical review on request.*

## The problem

Business data shows what moved together. Decisions need to know what happens if we act.

In the engine's retention benchmark, a simple comparison of customers who did and did not receive a retention offer suggests the offer **raised** churn by about 8 percentage points. The true effect, built into the simulation, is a 10-point **reduction**: the offer went to customers who were already about to leave. Across 500 simulated datasets the engine recovers an average of −9.97 pp.

## How it works

```
spec → data → causal graph → identification → estimation → refutation → evidence record → human review
```

- **Spec:** one YAML file with the question, the data, the role of each variable, and domain constraints (for example temporal order: a later event cannot cause an earlier one).
- **Causal graph:** given by experts, or discovered from the data under those constraints.
- **Identification:** which variables must be adjusted for, and which must be left alone (colliders).
- **Estimation:** doubly robust (AIPW) and double machine learning with cross-fitted ML models, with regression and weighting as baselines.
- **Refutation:** placebo treatments, noise and subset tests, negative controls, sensitivity analysis (robustness value).
- **Evidence record:** fingerprints of data, graph and spec, the assumptions, the results, and an append-only review log. Any past report can be re-rendered exactly as it was.

Four question types: effect of an intervention, ranking of drivers, attribution of a KPI shift between two periods, and what-if simulation.

## The design decision that matters most

An early version labelled a result "Supported" when the estimate was significant and survived the refutation tests. Monte Carlo testing with a **hidden** confounder showed the weakness: the estimate was off by 7.55 pp, its 95% interval never covered the truth, and the engine still called it "Supported" in 17 of 20 full runs.

I treated this as a defect and redesigned the verdict into three separate dimensions:

| dimension | question | evidence |
|---|---|---|
| Statistical evidence | Is the effect distinguishable from zero? | confidence interval |
| Estimate robustness | Does it survive perturbation? | refutation tests, overlap, sensitivity, estimator cross-check |
| Causal validity | Do the causal assumptions hold up? | negative controls, robustness value against the strongest measured confounder |

"Supported" now requires all three. Without a negative control, causal validity is explicitly *unverified* and the best available label is "Conditional". A negative control counts as evidence only if its whole confidence interval lies inside the equivalence margin.

The rerun was gated by acceptance criteria set in advance (20 full runs per scenario, n = 20,000 each):

| scenario | verdict after the redesign |
|---|---|
| hidden confounder, no negative control | Conditional 17, No detectable effect 3. Never Supported (previously Supported 17). |
| hidden confounder, with negative control | Challenged 20/20 |
| observed confounder, with negative control | Supported 17, Conditional 3. No false alarm. |
| observed confounder, no negative control | Conditional 20/20 |

The three Conditional results among the clean runs are the cost of not over-claiming: there the negative control's interval reaches past the margin, and the engine says so. Estimates, standard errors and intervals were identical to the previous version in all 40 runs; only the labels changed.

The same principle applies to any AI system in production: a confidence label is worth only as much as the evidence behind it, and that evidence has to be designed in and tested.

## Validation

Every check compares the engine with a known answer.

**1. Synthetic ground truth: 20/20 checks.** Effects with given and discovered graphs, adjustment-set selection, driver ranking, KPI-shift attribution, what-if simulation. Some checks are built to miss on purpose (for example adjusting for a collider) to show the bias the engine avoids.

**2. LaLonde (NSW job-training programme), the standard external benchmark.** A randomised experiment gives the true effect; the engine only sees the observational version, where the treated and comparison groups differ sharply.

| method | estimate (USD) | 95% interval |
|---|---:|---|
| experimental benchmark (truth) | 1,794 | |
| naive comparison | −8,498 | [−9,644, −7,351] |
| weighting only (poor overlap) | −3,495 | [−4,755, −2,235] |
| doubly robust, linear models | 1,550 | [216, 2,885] |
| doubly robust, gradient boosting | 1,826 | [247, 3,405] |

**3. Monte Carlo: does the reported uncertainty mean what it says?** Retention scenario, true effect −10.00 pp, n = 20,000 per dataset.

| | confounder measured (500 datasets) | confounder hidden (100 datasets) |
|---|---:|---:|
| mean estimate | −9.97 pp | −2.45 pp |
| bias | +0.03 pp | +7.55 pp |
| RMSE | 0.67 pp | 7.60 pp |
| 95% interval coverage | 95.0% | 0.0% |

With the confounder measured, the intervals cover the truth at their nominal rate, and the reported standard errors match the actual spread of the estimates (ratio 0.99). With it hidden, the data alone cannot reveal the bias. The verdict redesign exists so that the engine stops claiming otherwise.

## What this does not prove

- The example datasets other than LaLonde are synthetic. They test the machinery under assumptions the engine shares; LaLonde is the one external check.
- The test that matters next is a past intervention in real company data, held out and predicted before looking at the outcome. That has not been run yet.
- What-if intervals come from 20 bootstrap refits, so their tails are rough.

## Stack

Python · DoWhy · EconML · causal-learn · scikit-learn · statsmodels · pandas · FastAPI · Docker · Caddy (HTTPS, per-user login) · GitHub Actions (tests and container build on every push)

A web interface builds the spec in six steps, lets the user edit the causal graph edge by edge, queues runs with live progress and records the review sign-off. The command line and HTTP API run the same engine on the same files.

## My role

I defined the scope and the validation standard, found the verdict defect described above, specified the redesign and its acceptance criteria, and reviewed every validation result. The code was written with an AI coding assistant working to those specifications.
