---
name: prd-to-pipeline-plan
description: "Analyze a PRD file (XLSX, CSV) containing source onboarding requests and business rules, then produce a structured implementation plan for updating a target Dynamic Table. Use when: onboarding new sources, processing business requirements docs, planning pipeline changes from PRD files. Triggers: PRD, business requirements, source onboarding, pipeline plan, onboard source."
---

# PRD to Pipeline Plan

Turn product requirement documents into actionable Dynamic Table implementation plans. Surfaces assumptions and open questions instead of guessing.

## When to Use

- You receive an XLSX or CSV file with source onboarding requests or business rules
- You need to plan changes to an existing Dynamic Table (adding sources, new rules, schema changes)
- You want a structured gap analysis between a PRD and the current pipeline state

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Path to the PRD file (XLSX or CSV). May be multiple files. |
| `target_dynamic_table` | Yes | Fully qualified name of the target DT (e.g., `DB.SCHEMA.SILVER_TABLE`) |

If the user does not provide these, ask:
```
To generate a pipeline plan, I need:
1. **PRD file path** — the XLSX or CSV with requirements
2. **Target Dynamic Table** — fully qualified name (DB.SCHEMA.TABLE)
```

## Workflow

### Step 1: Parse the PRD

1. Read the PRD file(s) using the Read tool (supports XLSX and CSV natively)
2. Identify the document structure:
   - **Source onboarding sheets/rows**: new systems, contacts, timelines, data formats
   - **Business rules sheets/rows**: normalization, dedup, filtering, derived columns
   - **Column mapping sheets/rows**: source column → target column relationships
3. Extract all requirements into a working set

### Step 2: Inspect Current Pipeline State

1. Run `SHOW DYNAMIC TABLES LIKE '<table>' IN SCHEMA <db>.<schema>`
2. Run `SELECT GET_DDL('DYNAMIC_TABLE', '<fully_qualified_name>')` to get the current definition
3. Run `DESCRIBE DYNAMIC TABLE <fully_qualified_name>` to get the column list
4. Identify the current source tables by inspecting the DDL (look for FROM/UNION ALL references)

### Step 3: Gap Analysis

Compare PRD requirements against the current DT state:

1. **New sources**: Which source tables in the PRD are NOT yet in the DT's UNION ALL?
2. **New columns**: Does the PRD introduce fields not in the current Silver schema?
3. **New business rules**: What normalization, dedup, or transformation logic is new?
4. **Changed rules**: Do any PRD rules conflict with existing DT logic?
5. **Dependencies**: Are prerequisite tables/data available, or is something blocked?

### Step 4: Classify Decisions

For every requirement, classify it as one of:

| Classification | Meaning | Action |
|----------------|---------|--------|
| **Confirmed** | Clear owner, explicit decision documented in PRD | Implement as specified |
| **Inferred** | Not explicit but follows established patterns | State the inference, flag for review |
| **Open Question** | Ambiguous, conflicting, or missing decision | Do NOT implement — surface to user |

**Critical rule**: Never silently resolve an ambiguity. If two interpretations are possible, list both and ask.

**⚠️ STOP**: Present the gap analysis and classified decisions. Get user confirmation before proceeding to Step 5.

### Step 5: Produce the Implementation Plan

Generate a structured output (see Output section below) containing:

1. Summary of changes
2. Column mapping table for new sources
3. Business rules to implement (with SQL snippets)
4. Open questions requiring human decisions
5. Proposed DDL (compile-only validated with `only_compile: true`)

**⚠️ STOP**: Present the plan. Do NOT execute DDL without explicit user approval.

## Stopping Points

- ✋ After Step 4: Gap analysis and decision classification must be approved
- ✋ After Step 5: Implementation plan must be approved before any DDL execution

## Output

Always return ALL of the following sections:

### 1. New Source Systems

| Source System | Platform | Bronze Table | Status |
|---------------|----------|--------------|--------|
| ... | ... | ... | Ready / Blocked / Pending |

### 2. Column Mapping (per new source)

| Source Column | Silver Target Column | Transformation |
|---------------|---------------------|----------------|
| ... | ... | Direct / CASE / Rename / ... |

### 3. Business Rules to Implement

| Rule ID | Category | Description | SQL Pattern | Classification |
|---------|----------|-------------|-------------|----------------|
| ... | ... | ... | `CASE WHEN ...` | Confirmed / Inferred |

### 4. Open Questions

| # | Question | Why It Matters | Options | Owner |
|---|----------|---------------|---------|-------|
| 1 | ... | ... | A or B | ... |

### 5. Proposed DDL (compile-validated)

```sql
CREATE OR REPLACE DYNAMIC TABLE ...
```

Include a note on what changed vs. the current definition.

## Best Practices

### Surfacing Assumptions

- **Never normalize values without citing the rule**: If you map `POSTED` → `APPROVED`, cite which business rule (e.g., BR-001) authorizes it
- **Never add columns not in the PRD**: If the PRD doesn't mention a column, don't invent one
- **Never drop columns silently**: If a system-specific column is excluded, cite BR-008 or equivalent
- **Flag format inconsistencies**: If the same concept (e.g., payment terms) has different formats across sources, call it out even if one format "looks obvious"

### Open Questions vs. Decisions

Ask yourself for each mapping:
1. Is there an explicit documented decision with an owner? → Confirmed
2. Does it follow an established pattern already in the DT? → Inferred (flag it)
3. Neither? → Open Question (do not implement)

### DDL Validation

- Always use `only_compile: true` first to validate syntax
- Show the diff between current and proposed DDL
- Call out any TARGET_LAG, WAREHOUSE, or REFRESH_MODE changes separately

## Example Usage

**User prompt:**
```
Analyze assets/requirements.xlsx against COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
and give me an implementation plan for adding Baan and Workday sources.
```

**Expected workflow:**
1. Read `assets/requirements.xlsx` (source onboarding + business rules sheets)
2. `GET_DDL('DYNAMIC_TABLE', 'COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES')`
3. Identify that Baan and Workday are not in the current UNION ALL
4. Map Baan columns (BAN_INVOICE_ID → INVOICE_ID, BAN_VENDOR_DESC → VENDOR_NAME, etc.)
5. Map Workday columns (WD_INVOICE_ID → INVOICE_ID, WD_SUPPLIER_NAME → VENDOR_NAME, etc.)
6. Apply BR-001 (status normalization), BR-003 (Baan dedup), BR-008 (drop system columns)
7. Flag BR-005 (payment terms) as Open Question
8. Present gap analysis → get approval → produce DDL
