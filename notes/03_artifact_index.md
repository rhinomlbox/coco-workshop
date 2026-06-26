# PRD-Driven Pipeline Update: Artifact Index

## Files Produced This Session

| # | Path | Purpose |
|---|------|---------|
| 1 | `sql/02_silver_ap_invoices_proof.sql` | Baseline proof query (record counts by SOURCE_SYSTEM) |
| 2 | `sql/03_silver_ap_invoices_prd_update.sql` | Updated Dynamic Table DDL adding Baan and Workday sources |
| 3 | `notes/02_prd_change_plan.md` | Full change plan: mappings, open questions, assumptions, validation queries |
| 4 | `.cortex/skills/prd-to-pipeline-plan/SKILL.md` | Reusable project skill for turning PRD files into pipeline plans |

## Input PRDs (unchanged, in `assets/`)

| File | Contains |
|------|----------|
| `assets/sample_business_requirements_source_onboarding.csv` | New source system requests (Baan, Workday) with contacts, timelines, blockers |
| `assets/sample_business_requirements_business_rules.csv` | Business rules (BR-001 through BR-010): normalization, dedup, retention |
| `assets/sample_business_requirements_column_mapping.csv` | Column-level mapping from Baan/Workday to Silver schema with transforms |

## How to Review

1. Read `notes/02_prd_change_plan.md` for the full gap analysis and decision log
2. Review the DDL in `sql/03_silver_ap_invoices_prd_update.sql`
3. Check the two open questions (payment terms normalization, Baan dedup scope) before promoting to production

## How to Rerun Validation

```sql
-- After deploying the DDL, run these checks:

-- Count by source (expect SAP=15, ORACLE=15, BAAN=10, WORKDAY=10)
SELECT SOURCE_SYSTEM, COUNT(*) AS RECORD_COUNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM ORDER BY SOURCE_SYSTEM;

-- Status values (expect only APPROVED, PENDING)
SELECT DISTINCT APPROVAL_STATUS
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES ORDER BY 1;

-- Baan dedup (expect 0 rows)
SELECT INVOICE_NUMBER, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER HAVING CNT > 1;

-- Surrogate key uniqueness (expect 0 rows)
SELECT INVOICE_SK, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY INVOICE_SK HAVING CNT > 1;
```

## How to Reuse the PRD Evaluator Skill

The project skill at `.cortex/skills/prd-to-pipeline-plan/SKILL.md` can be invoked for future PRD files:

```
Analyze <path-to-prd> against <DB.SCHEMA.TARGET_DT>
and give me an implementation plan.
```

It will:
1. Parse the PRD (XLSX or CSV)
2. Inspect the current Dynamic Table DDL
3. Produce a gap analysis with Confirmed / Inferred / Open Question classifications
4. Generate compile-validated DDL
5. Stop for approval before executing anything
