# Semantic View Audit Report: SV_AP_ANALYTICS

**Target:** `COCO_WORKSHOP.PIPELINE_LAB.SV_AP_ANALYTICS`
**Audited:** 2026-06-25
**YAML Source:** `semantic_view_20260625/creation/SV_AP_ANALYTICS_semantic_model.yaml`

---

## Summary

| Category | Pass | Warn | Info |
|----------|------|------|------|
| Documentation | 17/17 | 0 | 0 |
| Naming Conventions | 17/17 | 0 | 0 |
| Data Types | 17/17 | 0 | 0 |
| Synonyms | - | 2 | 0 |
| Filters | 4/4 | 0 | 1 |
| Verified Queries | 5/5 compile | 1 | 2 |
| Inconsistencies | 0 found | 0 | 0 |
| Duplicates | 0 found | 0 | 0 |
| Missing Relationships | N/A (single table) | 0 | 0 |

---

## PASS - What's Working Well

| Check | Status |
|-------|--------|
| All 13 dimensions have descriptions | PASS |
| All 3 time dimensions have descriptions | PASS |
| All 1 fact has description | PASS |
| All 4 named filters have descriptions | PASS |
| Model-level description present and descriptive | PASS |
| No special characters in names | PASS |
| Consistent UPPER_SNAKE_CASE naming | PASS |
| All data types declared | PASS |
| Time dimensions correctly typed (DATE, TIMESTAMP_NTZ) | PASS |
| INVOICE_AMOUNT correctly classified as fact (not dimension) | PASS |
| All 5 VQRs compile against the physical table | PASS |
| No conflicting descriptions between model-level and column-level | PASS |
| No duplicate instructions between columns | PASS |

---

## WARNING - Should Address

| # | Severity | Issue | Location | Suggestion |
|---|----------|-------|----------|------------|
| 1 | WARNING | Synonym "system" is overly generic | `SOURCE_SYSTEM.synonyms` | Remove "system" or replace with "source system". Could match unrelated queries. |
| 2 | WARNING | Synonym "total" on INVOICE_AMOUNT may conflict with COUNT | `INVOICE_AMOUNT.synonyms` | Replace "total" with "total amount" or "total spend". "Total invoices" likely means COUNT, not SUM. |

---

## INFO - Recommendations for Improvement

| # | Category | Recommendation | Rationale |
|---|----------|---------------|-----------|
| 1 | Missing metrics | Add explicit metrics section (total_spend, invoice_count, average_invoice_amount) | Gives Cortex Analyst unambiguous aggregation patterns. |
| 2 | VQR coverage gap | No VQR for "overdue invoices" despite having a named filter | Common question pattern needs grounding. |
| 3 | VQR coverage gap | No VQR demonstrating the high_value filter | A VQR like "Which invoices exceed $500,000?" would improve accuracy. |
| 4 | Missing synonyms | CREATED_AT has no synonyms | Users may say "creation date" or "record created". |
| 5 | Single fact limits aggregation clarity | Only INVOICE_AMOUNT; users will also ask for COUNT(*) | Add as a metric or second fact. |

---

## Suggested Fixes

### 1. Remove ambiguous synonyms (WARNING)

```yaml
# SOURCE_SYSTEM: remove "system", keep "erp system" and "source"
synonyms:
  - erp system
  - source

# INVOICE_AMOUNT: replace "total" with "total amount"
synonyms:
  - amount
  - spend
  - invoice value
  - total amount
```

### 2. Add metrics section (INFO - high value)

```yaml
metrics:
  - name: total_spend
    expr: SUM(INVOICE_AMOUNT)
    description: Sum of invoice amounts. Always group by CURRENCY_CODE when using this metric.
  - name: invoice_count
    expr: COUNT(*)
    description: Number of invoices.
  - name: average_invoice_amount
    expr: AVG(INVOICE_AMOUNT)
    description: Average invoice amount. Group by CURRENCY_CODE for meaningful results.
```

### 3. Add missing VQRs (INFO)

```yaml
- name: overdue_invoices
  question: Show me all overdue invoices
  sql: >
    SELECT INVOICE_NUMBER, VENDOR_NAME, INVOICE_AMOUNT, CURRENCY_CODE, DUE_DATE
    FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
    WHERE DUE_DATE < CURRENT_DATE() AND APPROVAL_STATUS = 'PENDING'
    ORDER BY DUE_DATE
  verified_at: 1782422034
  verified_by: Manual

- name: high_value_invoices
  question: Which invoices exceed $500,000?
  sql: >
    SELECT INVOICE_NUMBER, VENDOR_NAME, INVOICE_AMOUNT, CURRENCY_CODE, SOURCE_SYSTEM
    FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
    WHERE INVOICE_AMOUNT > 500000
    ORDER BY INVOICE_AMOUNT DESC
  verified_at: 1782422034
  verified_by: Manual
```

### 4. Add synonyms to CREATED_AT (INFO)

```yaml
- name: CREATED_AT
  expr: CREATED_AT
  data_type: TIMESTAMP_NTZ
  description: Timestamp when the invoice record was created in the source system (UTC).
  synonyms:
    - creation date
    - record created
```

---

## VQR Compile Check Results

| VQR Name | Compiles | Notes |
|----------|----------|-------|
| total_spend_by_vendor_12m | PASS | |
| invoice_count_by_month_and_unit | PASS | |
| top_10_vendors_unpaid | PASS | |
| invoices_by_source_system | PASS | |
| count_and_amount_by_status | PASS | |

---

## Next Steps

- [ ] Apply synonym fixes (WARNING items)
- [ ] Add metrics section
- [ ] Add overdue and high_value VQRs
- [ ] Add CREATED_AT synonyms
- [ ] Redeploy updated YAML to Snowflake
- [ ] Run VQR behavioral test (ask questions without hints)
