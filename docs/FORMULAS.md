# Formulas used in AJ Lending

Every formula below comes from the project's code. The file named under each heading is where it is used. Worked examples use the same numbers the code produces.

---

## 1. Probability of default (PD)

**Where:** `src/credit_risk/modeling.py` (model), `training/train.py` (training), the SageMaker endpoint (serving)

PD is the model's estimate of the chance that an applicant will not repay. It is not a hand-written formula. It comes from **XGBoost**, a model made of many small decision trees added together:

```
score_log_odds = tree_1(x) + tree_2(x) + ... + tree_400(x)
PD = 1 / (1 + e^(-score_log_odds))        (the logistic function)
```

- `x` is the applicant's input features (income, loan amount, late payments, and so on).
- Settings: 400 trees, depth 5, learning rate 0.05, 80% row and column sampling, `min_child_weight` 5, `reg_lambda` 1.
- **Monotonic constraints** (`src/credit_risk/schema.py`): for 13 inputs, the model is forced to move in one direction. More late payments, a higher debt-to-income ratio, or a higher refusal rate can only raise PD. A higher external score can only lower it. This stops advice like "3 more late payments lowers your risk".
- Data split: 70% train, 15% validation, 15% test, stratified by outcome (`random_state=42`).

## 2. Approve or decline (decision threshold)

**Where:** `src/credit_risk/modeling.py` → `decline_threshold`

The lender declines the riskiest 10% of applicants (`DECLINE_RATE = 0.10`, a policy choice):

```
threshold = the 90th percentile of PD on the validation set
decision  = APPROVE if PD < threshold, otherwise DECLINE
```

## 3. Credit score (300 to 850)

**Where:** `src/credit_risk/scorecard.py` → `aj_score`

This is the standard bank "scorecard scaling". PD is turned into odds, and the score moves by a fixed number of points each time the odds double:

```
odds   = (1 - PD) / PD                   (odds of repaying)
FACTOR = PDO / ln(2)  = 40 / 0.6931 = 57.708
score  = 700 + FACTOR × ln(odds / 50)
score  = clamp(score, 300, 850), rounded to a whole number
```

- Anchor: **700 points at odds of 50 to 1** (PD about 2%).
- **PDO = 40**: every 40 points, the odds of repaying double.
- PD is limited to the range 0.000001 to 0.999999 before the calculation.

**Worked examples**

| PD | Odds | Score |
|---|---|---|
| 2% | 49 : 1 | 699 |
| 5% | 19 : 1 | 644 |
| 10% | 9 : 1 | 601 |
| 20% | 4 : 1 | 554 |

**Going back from score to PD:** `PD = 1 / (1 + 50 × e^((score − 700) / 57.708))`. For a score of 600 this gives PD about 10.2%.

**Score bands:** 750 or more Excellent, 670 to 749 Good, 580 to 669 Fair, below 580 Poor.

The approval score shown to people is the same formula applied to the threshold PD (`aj_score(threshold)`).

This is AJ Lending's own score from its own model, not a FICO or bureau score.

## 4. Model quality metrics

**Where:** `src/credit_risk/metrics.py`

| Metric | Formula | Deployed model |
|---|---|---|
| AUC | Area under the ROC curve: the chance that a random defaulter gets a higher PD than a random non-defaulter | 0.741 |
| Gini | `2 × AUC − 1` | 0.481 |
| KS | `max(TPR − FPR)`: the biggest gap between the share of defaulters and non-defaulters caught at any cut-off | 0.364 |

## 5. Reason codes (SHAP)

**Where:** `src/credit_risk/explain.py`

SHAP splits a prediction into one contribution per feature, measured in log-odds, using XGBoost's exact TreeSHAP (`pred_contribs=True`):

```
score_log_odds = base_value + contribution_1 + contribution_2 + ... + contribution_n
```

The top reasons for a decline are the features with the **largest positive contributions** (those that pushed risk up). Up to 4 are shown, because the US rule (Regulation B) allows up to four reasons. Features that share one reason text (for example the external scores) are shown once.

## 6. Ratios derived from the application

**Where:** `src/credit_risk/applicant_input.py` and `src/credit_risk/derive.py` (matches `features.py`, checked by a test)

```
PCT_LATE_PAYMENTS    = late installments / past installments
AVG_DAYS_LATE        = (late installments × average days late when late) / past installments
PREV_REFUSAL_RATE    = previous refusals / previous applications
CREDIT_INCOME_RATIO  = loan amount / annual income
ANNUITY_INCOME_RATIO = yearly payment / annual income
CREDIT_TERM_MONTHS   = loan amount / payment per period
DEBT_TO_INCOME       = (existing bureau debt + loan amount) / annual income
```

If a value is missing or the divisor is zero, the ratio is left blank rather than guessed.

## 7. Monthly loan payment

**Where:** `src/credit_risk/loan_products.py` → `monthly_payment` (the web app's calculator uses the same formula)

The standard amortized payment:

```
r       = APR / 100 / 12                       (monthly rate)
payment = amount × r / (1 − (1 + r)^(−months))
payment = amount / months                      (if the rate is 0)
```

**Examples:** $20,000 personal loan at 12% for 36 months gives **$664.29 a month**. A $250,000 home loan at 6.75% for 360 months gives **$1,621.50 a month**.

AJ Lending's fictional rates and usual terms:

| Loan | APR | Usual term |
|---|---|---|
| Personal | 12.0% | 36 months |
| Education | 8.5% | 120 months |
| Home (mortgage) | 6.75% | 360 months |
| Business | 10.5% | 84 months |
| Car | 7.5% | 60 months |
| Credit card | 22.0% | n/a |

Loan type is **not** a model input. It matters because it sets the rate, which sets the payment, which changes the payment-to-income ratio that the model does use.

## 8. Existing loans

**Where:** `src/credit_risk/existing_loans.py`

```
monthly interest = balance × APR / 100 / 12
remaining days   = remaining months × 30.44
utilization      = balance / credit limit            (credit cards)
```

**Months left** at a fixed payment (solving the amortization formula for the number of months):

```
months_left = −ln(1 − balance × r / payment) / ln(1 + r)
months_left = balance / payment                      (if the rate is 0)
```

It returns "never" if `payment <= balance × r`, because the payment does not even cover the interest.

**Example:** $10,000 at 7.5% with a $400 payment has about **27.3 months** left. $20,000 at 12% costs **$200 in interest** in the first month.

When a loan has no known rate or payment, they are estimated from the table above and marked as estimates. Totals feed the model's debt inputs: total balance, monthly payment, monthly interest, and the overall card utilization `sum(card balances) / sum(card limits)`.

## 9. Path to approval (what-if)

**Where:** `src/credit_risk/counterfactual.py` and `src/credit_risk/derive.py` → `with_new_loan_amount`

For a declined application, the system tries smaller loans in 5% steps, from 95% down to 40% of the request:

```
new_amount  = requested × fraction
new_payment = old payment × (new_amount / old_amount)
```

The ratios are recalculated, the model scores the new loan, and the first amount where `PD < threshold` is suggested. If none works down to 40%, it says so. The web app and agent score all the options in **one** endpoint call.

## 10. Late-payment slider

**Where:** `src/credit_risk/applicant_input.py` → `late_payment_curve`, served by `lambda/api/handler.py`

The slider scores the applicant for k = 0 to 12 late installments among the next 12 payments, all in one endpoint call:

```
past_installments(k) = past_installments + 12
late_installments(k) = late_installments + k
avg_days_late        = at least 15 days when k > 0
PCT_LATE_PAYMENTS    = late_installments / past_installments    (section 6)
```

For each k, the model's PD is converted to a credit score (section 3) and a decision. The curve also reports the first k at which the decision becomes DECLINE. "Pay the next 12 on time" is the k = 0 point.

## 11. Fairness: four-fifths rule

**Where:** `src/credit_risk/fairness.py`

```
approval_rate(group)    = approved applicants in the group / all applicants in the group
disparate_impact_ratio  = lowest group approval rate / highest group approval rate
passes                  = ratio >= 0.8
```

Groups with fewer than 100 people are reported but left out of the ratio, so a handful of applicants cannot decide the result by chance. Gender and age bands (under 25, 25 to 40, 40 to 60, 60 and over) are used only to test fairness, never to score. Results: ratios of 0.632 before the fix, and 0.825 and 0.965 after.

## 12. Drift: Population Stability Index (PSI)

**Where:** `src/credit_risk/metrics.py` → `psi`, `monitoring/drift_lambda.py`

Compare the training data ("expected") with recent requests ("actual") using 10 buckets cut at the training data's percentiles:

```
PSI = sum over buckets of (actual% − expected%) × ln(actual% / expected%)
```

A PSI **above 0.25** is treated as a significant shift and triggers the CloudWatch alarm. Empty buckets are set to 0.000001 to avoid dividing by zero.

---

AJ Lending is fictional and the rates above are made up for this portfolio project. This document is a learning summary, not financial or legal advice.
