# PRD Change Plan: SILVER_AP_INVOICES

**PRD Source:** `assets/sample_business_requirements_column_mapping.csv`
**Target:** `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`
**Generated:** 2026-06-25

---

## 1. Summary of Requested Changes

The PRD adds **two new source systems** (Baan IV and Workday) to `SILVER_AP_INVOICES`, which currently unions only SAP and Oracle. Both bronze tables already exist in `COCO_WORKSHOP.SOURCE_DATA`.

| Change Type | Count |
|-------------|-------|
| New UNION ALL branches | 2 (Baan, Workday) |
| New columns to Silver schema | 0 (all map to existing columns) |
| Status normalization rules | 2 (Baan: POSTED→APPROVED; Workday: Approved→APPROVED, In Review→PENDING) |
| Columns to drop (not pass to Silver) | 2 (BAN_COMPANY, WD_TENANT_ID) |
| Dedup logic (Baan only) | 1 (QUALIFY ROW_NUMBER on INVOICE_NUMBER) |

No schema change to the Silver DT column list is required -- all Baan and Workday columns map to the existing 17 Silver columns.

---

## 2. Source-to-Silver Mapping Summary

### Baan IV → SILVER_AP_INVOICES

| Source Column | Silver Column | Transform | Classification |
|---------------|--------------|-----------|----------------|
| BAN_INVOICE_ID | INVOICE_ID | Direct | Confirmed |
| BAN_INVOICE_REF | INVOICE_NUMBER | Direct | Confirmed |
| BAN_VENDOR_CODE | VENDOR_ID | Direct | Confirmed |
| BAN_VENDOR_DESC | VENDOR_NAME | Direct | Confirmed |
| BAN_INV_DATE | INVOICE_DATE | Direct | Confirmed |
| BAN_PAY_DATE | DUE_DATE | Direct | Confirmed |
| BAN_AMOUNT | INVOICE_AMOUNT | Direct | Confirmed |
| BAN_CURR | CURRENCY_CODE | Direct | Confirmed |
| BAN_PAY_TERMS | PAYMENT_TERMS | **See Open Q #1** | Open |
| BAN_PO_REF | PO_NUMBER | Direct | Confirmed |
| BAN_LINE_DESC | LINE_DESCRIPTION | Direct | Confirmed |
| BAN_GL_CODE | GL_ACCOUNT | Direct | Confirmed |
| BAN_COST_CTR | COST_CENTER | Direct | Confirmed |
| BAN_STATUS | APPROVAL_STATUS | CASE (POSTED→APPROVED) | Confirmed |
| BAN_CREATED | CREATED_AT | Direct | Confirmed |
| BAN_COMPANY | — | DROP | Confirmed |

### Workday → SILVER_AP_INVOICES

| Source Column | Silver Column | Transform | Classification |
|---------------|--------------|-----------|----------------|
| WD_INVOICE_ID | INVOICE_ID | Direct | Confirmed |
| WD_INVOICE_NUM | INVOICE_NUMBER | Direct | Confirmed |
| WD_SUPPLIER_ID | VENDOR_ID | Direct | Confirmed |
| WD_SUPPLIER_NAME | VENDOR_NAME | Direct | Confirmed |
| WD_INVOICE_DATE | INVOICE_DATE | Direct | Confirmed |
| WD_DUE_DATE | DUE_DATE | Direct | Confirmed |
| WD_AMOUNT | INVOICE_AMOUNT | Direct | Confirmed |
| WD_CURRENCY | CURRENCY_CODE | Direct | Confirmed |
| WD_PAY_TERMS | PAYMENT_TERMS | **See Open Q #1** | Open |
| WD_PO_NUMBER | PO_NUMBER | Direct | Confirmed |
| WD_MEMO | LINE_DESCRIPTION | Direct | Confirmed |
| WD_LEDGER_ACCOUNT | GL_ACCOUNT | Direct | Confirmed |
| WD_COST_CENTER | COST_CENTER | Direct | Confirmed |
| WD_APPROVAL_STATUS | APPROVAL_STATUS | CASE (Approved→APPROVED, In Review→PENDING) | Confirmed |
| WD_CREATED_DATE | CREATED_AT | Direct | Confirmed |
| WD_TENANT_ID | — | DROP | Confirmed |

---

## 3. Open Questions and Assumptions

### Open Questions (DO NOT implement without decision)

| # | Question | Why It Matters | Options | Owner | PRD Ref |
|---|----------|---------------|---------|-------|---------|
| 1 | **Should Silver normalize payment terms?** | The current DT already normalizes Oracle `N30`→`NET30`. PRD says "keep as-is, normalize in Gold" but that conflicts with existing Oracle logic. | **A)** Revert Oracle normalization, pass all terms raw (Baan: `N30`, Workday: `Net 30`, SAP: `NET30`) and normalize in Gold. **B)** Normalize all sources to `NET30`/`NET60` at Silver (consistent but undecided). | Sarah Chen / David Kim | BR-005 |
| 2 | **Baan dedup: partition scope** | BR-003 says deduplicate on INVOICE_NUMBER. Must this run only within the Baan branch (to avoid collision with other systems that might reuse the same invoice number format)? | **A)** QUALIFY within Baan subquery only. **B)** QUALIFY after UNION ALL partitioned by SOURCE_SYSTEM + INVOICE_NUMBER. | Engineering | BR-003 |

### Assumptions (Inferred -- flagged for review)

| # | Assumption | Basis |
|---|-----------|-------|
| 1 | Baan dedup uses `QUALIFY ROW_NUMBER() OVER (PARTITION BY BAN_INVOICE_REF ORDER BY BAN_CREATED DESC) = 1` applied within the Baan branch before UNION ALL | BR-003 + pattern of isolating source-specific logic in its own branch |
| 2 | `SOURCE_SYSTEM` literal for Baan is `'BAAN'` (not `'BAAN IV'`) | BR-007 says use short identifiers: 'SAP', 'ORACLE', 'BAAN', 'WORKDAY' |
| 3 | For payment terms, pass through raw values (no normalization) for Baan and Workday pending decision on Open Q #1. The existing Oracle normalization is left in place to avoid breaking current behavior. | Least-change principle while decision is pending |

---

## 4. DDL Delta Plan

**What changes vs. current:**
- Add UNION ALL branch for Baan (with dedup QUALIFY)
- Add UNION ALL branch for Workday
- Baan: CASE on BAN_STATUS for approval normalization
- Workday: CASE on WD_APPROVAL_STATUS for approval normalization
- Payment terms: passed through raw (pending decision)

**Status:** Compile-validated successfully.

```sql
CREATE OR REPLACE DYNAMIC TABLE COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
  TARGET_LAG = '1 hour'
  WAREHOUSE = COCO_WORKSHOP_WH
AS
SELECT
    MD5(SOURCE_SYSTEM || '|' || INVOICE_ID) AS INVOICE_SK,
    SOURCE_SYSTEM,
    INVOICE_ID,
    INVOICE_NUMBER,
    VENDOR_ID,
    VENDOR_NAME,
    INVOICE_DATE,
    DUE_DATE,
    INVOICE_AMOUNT,
    CURRENCY_CODE,
    PAYMENT_TERMS,
    PO_NUMBER,
    LINE_DESCRIPTION,
    GL_ACCOUNT,
    COST_CENTER,
    APPROVAL_STATUS,
    CREATED_AT
FROM (
    -- SAP (unchanged)
    SELECT
        'SAP' AS SOURCE_SYSTEM,
        INVOICE_ID, INVOICE_NUMBER, VENDOR_ID, VENDOR_NAME,
        INVOICE_DATE, DUE_DATE, INVOICE_AMOUNT, CURRENCY_CODE,
        PAYMENT_TERMS, PO_NUMBER, LINE_DESCRIPTION, GL_ACCOUNT,
        COST_CENTER, APPROVAL_STATUS, CREATED_AT
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_SAP_AP_INVOICES

    UNION ALL

    -- Oracle (unchanged)
    SELECT
        'ORACLE' AS SOURCE_SYSTEM,
        INV_ID, INV_NUM, SUPPLIER_ID, SUPPLIER_NAME,
        INV_DATE, PAYMENT_DUE_DATE, TOTAL_AMOUNT, CURRENCY,
        CASE TERMS_CODE WHEN 'N30' THEN 'NET30' WHEN 'N60' THEN 'NET60' ELSE TERMS_CODE END,
        PURCHASE_ORDER, DESCRIPTION, ACCOUNT_CODE,
        DEPT_CODE,
        CASE STATUS WHEN 'VALIDATED' THEN 'APPROVED' ELSE STATUS END,
        CREATION_DATE
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_ORACLE_AP_INVOICES

    UNION ALL

    -- Baan (NEW — with dedup per BR-003)
    SELECT
        'BAAN' AS SOURCE_SYSTEM,
        BAN_INVOICE_ID,
        BAN_INVOICE_REF,
        BAN_VENDOR_CODE,
        BAN_VENDOR_DESC,
        BAN_INV_DATE,
        BAN_PAY_DATE,
        BAN_AMOUNT,
        BAN_CURR,
        BAN_PAY_TERMS,
        BAN_PO_REF,
        BAN_LINE_DESC,
        BAN_GL_CODE,
        BAN_COST_CTR,
        CASE BAN_STATUS
            WHEN 'POSTED' THEN 'APPROVED'
            WHEN 'APPROVED' THEN 'APPROVED'
            WHEN 'PENDING' THEN 'PENDING'
            ELSE BAN_STATUS
        END,
        BAN_CREATED
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_BAAN_AP_INVOICES
    QUALIFY ROW_NUMBER() OVER (PARTITION BY BAN_INVOICE_REF ORDER BY BAN_CREATED DESC) = 1

    UNION ALL

    -- Workday (NEW)
    SELECT
        'WORKDAY' AS SOURCE_SYSTEM,
        WD_INVOICE_ID,
        WD_INVOICE_NUM,
        WD_SUPPLIER_ID,
        WD_SUPPLIER_NAME,
        WD_INVOICE_DATE,
        WD_DUE_DATE,
        WD_AMOUNT,
        WD_CURRENCY,
        WD_PAY_TERMS,
        WD_PO_NUMBER,
        WD_MEMO,
        WD_LEDGER_ACCOUNT,
        WD_COST_CENTER,
        CASE WD_APPROVAL_STATUS
            WHEN 'Approved' THEN 'APPROVED'
            WHEN 'In Review' THEN 'PENDING'
            ELSE UPPER(WD_APPROVAL_STATUS)
        END,
        WD_CREATED_DATE
    FROM COCO_WORKSHOP.SOURCE_DATA.BRONZE_WORKDAY_AP_INVOICES
);
```

---

## 5. Validation Queries (run after implementation)

```sql
-- 1. Record count by source (expect: SAP=15, ORACLE=15, BAAN=10, WORKDAY=10 → 50 total)
SELECT SOURCE_SYSTEM, COUNT(*) AS RECORD_COUNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM
ORDER BY SOURCE_SYSTEM;

-- 2. Confirm no VALIDATED/POSTED/In Review leaked through
SELECT DISTINCT APPROVAL_STATUS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
ORDER BY APPROVAL_STATUS;
-- Expected: APPROVED, PENDING only

-- 3. Confirm Baan dedup worked (no duplicate INVOICE_NUMBERs within BAAN)
SELECT INVOICE_NUMBER, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER
HAVING CNT > 1;
-- Expected: 0 rows

-- 4. Confirm dropped columns are not present
DESCRIBE DYNAMIC TABLE COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES;
-- Should NOT contain BAN_COMPANY, WD_TENANT_ID, SAP_COMPANY_CODE, etc.

-- 5. Confirm surrogate key uniqueness
SELECT INVOICE_SK, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY INVOICE_SK
HAVING CNT > 1;
-- Expected: 0 rows
```

---

## Next Steps

- Resolve Open Question #1 (payment terms normalization) before executing DDL
- Confirm Open Question #2 (Baan dedup scope — currently implemented within Baan branch only)
- Once decisions are made, execute the DDL and run the validation queries above
