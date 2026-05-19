# Claims Analytics Pipeline — Documentation

## Overview

This Lakeflow Designer pipeline builds a **curated claims analytics dataset** by joining three raw data sources (claims, payments, policies), enriching them with derived features, and writing the result to a Unity Catalog table for downstream ML and BI use.

**Output Table:** `zurich_lcnc_rfp_catalog.pc_demo.curated_claims_analytics_designer`

---

## Architecture Diagram

```
claims.parquet ──► claims_enriched ──┐
                                     ├──► claims_with_payments ──┐
payments.parquet ──► payments_agg ───┘                           │
                                                                 ├──► claims_full ──► claims_curated ──┬──► OUTPUT TABLE
policies.parquet ────────────────────────────────────────────────┘                                    ├──► claim count (aggregate)
                                                                                                      └──► linear_regression (Python)
```

---

## Data Sources

| Source | Format | Path |
| --- | --- | --- |
| **claims** | Parquet | `/Volumes/zurich_lcnc_rfp_catalog/pc_demo/raw/claims/claims.parquet` |
| **payments** | Parquet | `/Volumes/zurich_lcnc_rfp_catalog/pc_demo/raw/payments/payments.parquet` |
| **policies** | Parquet | `/Volumes/zurich_lcnc_rfp_catalog/pc_demo/raw/policies/policies.parquet` |

---

## Transformation Steps

### 1. claims_enriched (Transform)

Derives time-based features from raw claim dates:

| Derived Column | Logic |
| --- | --- |
| `reporting_delay_days` | `DATEDIFF(claim_report_date, claim_date)` |
| `claim_month` | `MONTH(claim_date)` |
| `claim_year` | `YEAR(claim_date)` |

All original claim columns are passed through.

---

### 2. payments_agg (Aggregate)

Aggregates payment records to one row per claim:

| Column | Aggregation |
| --- | --- |
| `total_payment_amount` | `SUM(payment_amount)` |
| `payment_count` | `COUNT(payment_id)` |

**Group By:** `claim_id`

---

### 3. claims_with_payments (Join)

Combines enriched claims with aggregated payments.

| Property | Value |
| --- | --- |
| Join Type | LEFT |
| Condition | `left.claim_id = right.claim_id` |
| Left Input | `claims_enriched` |
| Right Input | `payments_agg` |

Ensures all claims are retained even if no payments exist yet.

---

### 4. claims_full (Join)

Enriches claims with policy-level attributes.

| Property | Value |
| --- | --- |
| Join Type | LEFT |
| Condition | `left.policy_id = right.policy_id` |
| Left Input | `claims_with_payments` |
| Right Input | `policies` |

Adds: `policy_type`, `premium_amount`, `coverage_limit`, `deductible`, `region`, `risk_score`

---

### 5. claims_curated (Transform)

Final column selection and null handling:

- Selects all business-relevant columns
- `COALESCE(payment_count, 0)` — fills NULL (no payments) with 0
- `COALESCE(total_payment_amount, 0.0)` — fills NULL with 0.0

---

## Output

### Primary Output Table

**`zurich_lcnc_rfp_catalog.pc_demo.curated_claims_analytics_designer`**

| Column | Description |
| --- | --- |
| `claim_id` | Unique claim identifier |
| `policy_id` | Associated policy |
| `policy_type` | Policy category (from policies) |
| `premium_amount` | Annual premium (from policies) |
| `coverage_limit` | Max coverage (from policies) |
| `deductible` | Policy deductible (from policies) |
| `region` | Geographic region (from policies) |
| `risk_score` | Computed risk score (from policies) |
| `reporting_delay_days` | Days between incident and report (derived) |
| `claim_month` | Month of claim (derived) |
| `claim_year` | Year of claim (derived) |
| `claim_date` | Date of incident |
| `claim_report_date` | Date claim was reported |
| `claim_close_date` | Date claim was closed |
| `claim_amount` | Claimed amount |
| `paid_amount` | Amount paid out |
| `days_to_settle` | Days to resolve |
| `incident_severity` | Severity category (Low/Medium/High/Severe) |
| `claim_status` | Current status |
| `payment_count` | Number of payments made (from payments) |
| `total_payment_amount` | Sum of all payments (from payments) |

---

## Analytical Branches

### Claim Count Aggregate

Groups curated data by `region`, `policy_type`, and `incident_severity` to count claims per segment. Useful for BI dashboards and regional risk assessment.

### Linear Regression (Python)

Trains a scikit-learn `LinearRegression` model to predict `claim_amount` from:
- **Numeric features:** premium_amount, coverage_limit, deductible, reporting_delay_days, claim_month, claim_year, days_to_settle, payment_count, risk_score
- **Categorical features:** policy_type, region, incident_severity (one-hot encoded)

Outputs model coefficients with R² and RMSE metrics.

---

## Downstream Consumers

- **ML Notebook:** `Insurance Claims DS Demo - High Severity Prediction` — uses this table for classification model training
- **Lakehouse Monitor:** Tracks drift and performance on predictions derived from this dataset
- **Serving Endpoint:** `high-severity-claims-endpoint` — real-time scoring using features from this pipeline

---

## Maintenance Notes

- Pipeline reads from Unity Catalog Volumes (no external credentials needed)
- All joins are LEFT to preserve claim completeness
- NULL payment fields are coalesced to 0 — downstream models can safely assume no missing values in payment columns
- ~600K rows in output table (as of May 2026)
