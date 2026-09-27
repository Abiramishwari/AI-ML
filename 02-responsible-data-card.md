# Responsible Data Card — Customer Churn Training Data

*(`customer-churn-training.csv`, 12 rows, 7 columns)*

## Dataset purpose
**May support:** a first-pass, exploratory look at which easily-observed account signals (login recency, plan tier, support contacts, tenure, spend) correlate with subscription cancellation, in order to prioritize which customers a retention team calls first.

**Must not be used to:** set individual prices or discounts algorithmically, deny service, make employment or credit-adjacent decisions, or as the sole justification for account suspension. Given the dataset's size (see below), it also **must not be used to ship a production model as-is** — it is only suitable for problem framing and pipeline prototyping.

## Provenance and permission
- **Origin is undocumented.** The file has no metadata on when, how, or from what system it was extracted, and no confirmation of customer consent for this specific analytical use.
- Values are round numbers (₹499, ₹799, ₹999, ₹1299, ₹1499) that look like fixed plan-tier prices rather than raw billing extracts, and the perfect class separation described below is consistent with **hand-constructed or synthetic sample data** rather than a live production export.
- **Action item before any real use:** confirm with the data owner (a) the source system, (b) the extraction date, (c) whether the organization's privacy notice / terms of service cover this use, and (d) any retention limits on customer-level data.

## Population and representation
- Only **12 customers** are represented — far too small to support any claim about the wider customer base.
- Three plan tiers appear (Basic, Standard, Pro) with only 4 customers per tier on average; no tier has enough rows to assess separately.
- No geography, demographics, acquisition channel, or device/platform fields are present, so it's not possible to check whether any customer segment is over- or under-represented.
- `customer_id` values (C001–C012) are sequential and likely a proxy for signup order — if used as a feature this would implicitly encode "early adopter" rather than anything causal.

## Features and target
| Column | Description | Concerns |
|---|---|---|
| `customer_id` | Row identifier | Direct identifier; must be dropped or hashed before any model training or sharing, and access to the join key restricted. |
| `tenure_months` | Months since signup | No leakage concern. |
| `support_tickets` | Count of support tickets | Could reflect frustration (predictive) or simply engagement (not predictive) — direction of effect needs domain input. |
| `monthly_spend_inr` | Monthly spend | Can act as a **proxy for a customer's income/socioeconomic status**; treat as a sensitive-adjacent feature and monitor for disparate impact if ever combined with demographic data. |
| `last_login_days` | Days since last login | **Likely leakage risk.** If this is measured close to (or after) the churn decision, low recent engagement may be a symptom of having already decided to leave, not an early warning sign. The label window and this feature's observation window must be pinned down before trusting it. |
| `plan_type` | Basic / Standard / Pro | **Perfectly separates the target in this file** (all Basic customers churned, no Standard/Pro customer did). That is a strong signal of either a toy/synthetic dataset or an undocumented policy (e.g., Basic tier being sunset) rather than a generalizable pattern — flag for the data owner. |
| `churned` | Target label | Label window (churn within how many days, and measured from when) is undocumented. |

## Quality checks
- **Missingness:** none — all 7 columns are fully populated across all 12 rows.
- **Duplicates:** none — all 12 `customer_id` values are unique.
- **Class balance:** 5 churned / 7 retained (41.7% churn rate). This is unusually high for subscription churn and is itself a signal the sample is not representative of a real customer base.
- **Train/test separation:** not possible with 12 rows — any split would leave single-digit rows per class. The accompanying baseline notebook uses leave-one-out cross-validation for this reason, and even that should be read as illustrative, not as a real performance estimate.
- **Outliers:** none of the numeric columns show implausible values (tenure 1–30 months, 0–5 support tickets, spend ₹499–₹1499, last login 1–30 days).

## Risks and safeguards
See the accompanying risk register (`03-risk-register.md`) for the full list with likelihood, impact, and mitigations. Headline risks:
- **Bias/generalization risk:** conclusions from 12 rows will not hold on the real customer base. *Safeguard:* block this dataset from any production training pipeline; require a minimum row count and time span before retraining.
- **Leakage risk:** `last_login_days` (and possibly `plan_type`) may not have been available, in their current form, at the actual decision time. *Safeguard:* confirm feature timestamps against the label window before reuse.
- **Privacy risk:** `customer_id` is a direct identifier. *Safeguard:* strip or hash before any modeling, logging, or sharing outside the immediate team.
- **Misuse risk:** a model this confident (100% baseline accuracy) invites overtrust. *Safeguard:* require sign-off from a second reviewer before any churn score is used for a customer-facing action.

## Intended evaluation
- **Baseline:** majority-class predictor (58.3% accuracy on this file) and the two single-feature business rules described in the problem-framing memo.
- **Performance:** accuracy, precision, recall, and F1, always reported alongside the baseline they're being compared to — never in isolation.
- **Calibration:** not evaluable meaningfully at n=12; flag as required once a larger dataset is available.
- **Fairness:** once demographic or plan-tier-independent segment data exists, compare false-negative rates across segments (a customer who is missed by the model is the costlier error here).
- **Error analysis:** on any future larger dataset, manually review every false negative (missed churner) and false positive (unnecessarily contacted customer) before deployment sign-off.
