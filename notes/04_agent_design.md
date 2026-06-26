# Cortex Agent Design: AP Invoices Agent

**Target Table:** `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`
**Status:** Draft -- pending additional details before creation
**Generated:** 2026-06-25

---

## Primary Audience

Finance operations and procurement teams -- specifically AP analysts, controllers, and procurement managers who need self-service answers about invoice volumes, vendor spend, aging, and approval bottlenecks across all source ERP systems.

---

## Top Five Business Questions

| # | Question | Why It Matters |
|---|----------|---------------|
| 1 | What is the total spend by vendor? | Core procurement visibility -- identifies top suppliers for negotiation leverage |
| 2 | How many invoices are pending approval, and from which systems? | Surfaces approval bottlenecks that delay payments and damage vendor relationships |
| 3 | What is our monthly spend trend by source system? | Tracks whether new source integrations are flowing correctly and spend is growing/shrinking |
| 4 | Which invoices are overdue (DUE_DATE < today)? | Cash management and vendor relationship risk |
| 5 | What is the average invoice amount by currency? | Helps Treasury plan FX exposure and identifies outlier invoices |

---

## Guardrails

| Guardrail | Implementation | Rationale |
|-----------|---------------|-----------|
| Single-table scope | Query ONLY `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES` | Prevents hallucinated joins to tables the agent doesn't understand |
| No currency conversion | "Do NOT convert currencies. Report amounts in their original CURRENCY_CODE. Currency conversion happens in the Gold layer." | BR-002 -- avoids incorrect FX assumptions |
| No GL/cost center interpretation | "Do NOT interpret GL_ACCOUNT or COST_CENTER codes. They vary by source system and have no unified mapping at this layer." | BR-006 -- prevents false groupings |
| Clarification on ambiguous time ranges | "If the user says 'recent' without a date range, ask which time window they mean." | Prevents silent assumptions about recency |
| Amount thresholds | "If results include invoices over $500,000 USD equivalent, note that these are flagged for manual review per company policy." | BR-004 -- keeps finance aware of high-value items |
| No data modification | Agent has read-only SELECT access only | Standard security posture |

---

## Semantic Descriptions

These descriptions help the agent select the right columns and interpret them correctly.

### Table Description

```
Curated accounts payable invoice data harmonized from four ERP source systems
(SAP, Oracle, Baan, Workday). One row per unique invoice. Contains invoice
amounts in original transaction currency (no FX conversion). Refreshed hourly
via Dynamic Table. Use this table for all AP invoice analytics.
```

### Column Descriptions

| Column | Description |
|--------|-------------|
| INVOICE_SK | MD5 surrogate key (SOURCE_SYSTEM + INVOICE_ID). Use for uniqueness checks, not for display. |
| SOURCE_SYSTEM | Originating ERP system. Values: SAP, ORACLE, BAAN, WORKDAY. Use to filter or group by system. |
| INVOICE_ID | Source-native invoice identifier. Unique within a SOURCE_SYSTEM but NOT globally unique alone. |
| INVOICE_NUMBER | Business-facing invoice reference number. Used in vendor communications and payment matching. |
| VENDOR_ID | Source-native vendor/supplier code. Format varies by system (V-NNNN, S-NNNN, BV-NNN, WS-NNN). |
| VENDOR_NAME | Human-readable vendor/supplier name. Use this for display and GROUP BY in spend analysis. |
| INVOICE_DATE | Date the invoice was issued by the vendor. Use for trend analysis and time-based filtering. |
| DUE_DATE | Date payment is due. Compare to CURRENT_DATE() to find overdue invoices. |
| INVOICE_AMOUNT | Invoice total in the original transaction currency (see CURRENCY_CODE). Do NOT sum across currencies without conversion. |
| CURRENCY_CODE | ISO 4217 currency code (USD, EUR, GBP). Always filter or group by currency when aggregating amounts. |
| PAYMENT_TERMS | Payment terms as stored in source. Values vary: NET30, NET60, N30, N60, Net 30, Net 60. Not yet normalized. |
| PO_NUMBER | Purchase order reference. May be NULL for service invoices without a PO. |
| LINE_DESCRIPTION | Free-text description of the invoice line item. Useful for keyword search but not structured. |
| GL_ACCOUNT | General ledger account code from source system. Format varies by system. Do not cross-reference across systems. |
| COST_CENTER | Organizational cost center from source system. Format varies by system. Do not cross-reference across systems. |
| APPROVAL_STATUS | Normalized approval status. Values: APPROVED (ready for payment) or PENDING (awaiting approval). |
| CREATED_AT | Timestamp when the invoice record was created in the source system (UTC). |

---

## Data Profile (as of 2026-06-25)

- **Total rows:** 50
- **Source systems:** SAP (15), ORACLE (15), BAAN (10), WORKDAY (10)
- **Currencies:** USD, EUR, GBP
- **Statuses:** APPROVED, PENDING
- **Vendors:** 28 distinct
- **Date range:** 2025-01-10 to 2025-06-01
- **Payment terms (raw):** NET30, NET60, N30, N60, Net 30, Net 60

---

## Assumptions Requiring Engineering Review

| # | Assumption | Impact |
|---|-----------|--------|
| 1 | Agent queries only SILVER_AP_INVOICES -- no joins to bronze or dimension tables | Limits depth but keeps answers grounded. If vendor master or CoA lookups are needed later, a Gold table or second tool would be required. |
| 2 | PAYMENT_TERMS is not normalized -- the agent cannot reliably filter on terms without knowing all variants | Consider adding a computed PAYMENT_DAYS column (integer) in a future iteration for clean filtering. |
| 3 | Cross-currency aggregation is explicitly blocked | Users asking "total spend" without specifying currency will get per-currency breakdowns, which may feel verbose but is correct. |

---

## Next Steps

- [ ] Finalize additional details (instructions tone, response format preferences, etc.)
- [ ] Create semantic model YAML for Cortex Analyst tool
- [ ] Create the agent in Snowflake
- [ ] Build evaluation dataset from AGENT_EVAL_SET table
- [ ] Run baseline evaluation
