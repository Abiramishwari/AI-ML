# ML Problem Framing Memo — Customer Churn

## 1. The decision this model is meant to support
**Decision:** Which active customers should the retention team proactively contact (call, email, or offer a discount/incentive) *before* they cancel their subscription.

**What the model must NOT decide:**
- It must not automatically cancel, suspend, or restrict a customer's account.
- It must not set individualized pricing or determine credit terms.
- It must not be the sole basis for account-level actions with legal or financial consequence (e.g., denying a refund) — a human reviews any action beyond a routine retention outreach.

## 2. Prediction target
**Target column:** `churned` (1 = customer churned, 0 = customer retained).

Before training on this further, the label definition needs to be nailed down with the business owner:
- Over what horizon is "churned" measured (e.g., cancelled within 30 days of the snapshot date, or any time historically)?
- Is churn "voluntary cancellation" only, or does it also include non-renewal, downgrade, or payment failure?

This dataset does not document the label window — that is flagged as an open item in the data card and risk register below.

## 3. Unit of observation
One row = one customer, observed at a single point in time ("snapshot"), with features describing their state as of that snapshot and a label describing whether they churned afterward.

**Open question:** the current file has exactly one row per customer with no snapshot date. In production this needs to become a **recurring snapshot** (e.g., one row per customer per billing cycle) so the model can be scored regularly, not just once.

## 4. Action window
Proposed: **score customers monthly; flag anyone predicted to churn in the following 30 days** so retention outreach has time to act before the cancellation would occur. This window should be validated against how far in advance the retention team can realistically intervene (e.g., contract terms, notice periods).

## 5. Non-ML baselines (must beat these before shipping any model)
Three baselines were computed directly on the training file (see the accompanying notebook for full code and output):

| Baseline | Rule | Accuracy | Precision | Recall |
|---|---|---|---|---|
| Majority class | Always predict "not churned" | 58.3% | 0.0 | 0.0 |
| Business rule A | `last_login_days > 10` → churn | 100% | 1.0 | 1.0 |
| Business rule B | `plan_type == "Basic"` → churn | 100% | 1.0 | 1.0 |
| Logistic regression (leave-one-out CV) | full feature set | 91.7% | 0.83 | 1.0 |

**Key finding:** on this specific dataset, two single-variable business rules already reach perfect accuracy — and they outperform the logistic regression baseline. This is the single most important output of this exercise, and it drives the recommendation below.

## 6. Recommendation
1. **Do not conclude "ML works" from this file.** With only 12 rows and two features that separate the classes perfectly, this is almost certainly a small illustrative/synthetic sample, not a representative production dataset. A churn rate of 41.7% is also far higher than is typical for most subscription businesses, another sign this sample isn't representative.
2. **Before any ML investment**, deploy and monitor the simple rule (`last_login_days > 10`, or a similarly cheap combination) as the shipped baseline. It is transparent, auditable, and costs nothing to maintain.
3. **Only revisit ML** once a real historical dataset (hundreds+ rows, spanning multiple months, with a clearly defined label window) is available, and the simple rule's live performance is measured and found wanting.
4. Investigate whether `last_login_days` and `plan_type` are **actually available at decision time** or whether they leak information from close to (or after) the churn event itself — see the risk register.

## 7. Success metrics (for whichever approach ships)
- **Business metric:** net retained revenue after accounting for outreach cost and discount cost, measured over a rolling 90-day window.
- **Model/rule metric:** recall on churners weighted higher than precision (missing a churner is costlier than one unnecessary outreach), but tracked alongside a cap on false-positive rate so outreach costs stay bounded.
