# Data Integrity Controls

## Overview

Data integrity in a BI platform migration means the numbers in the new system match the numbers in the source system, the security rules produce identical row sets, and any known discrepancy is documented and accepted rather than hidden.

This document describes the controls architecture used in this framework: how issues are detected, how they are classified, how they are resolved, and how the system prevents regressions.

## Control layers

### Layer 1: Pre-migration inventory

Before converting any report, the source system is fully inventoried.

- `sql/obiee_source/extract_rpd_metadata.sql` extracts every table, column, join, and cardinality from the OBIEE RPD
- `sql/obiee_source/extract_catalog_security.sql` extracts every security rule, group assignment, and initialization block
- `sql/obiee_source/subject_area_inventory.sql` produces the full report inventory with source subject area and column counts

The inventory is the baseline. Any discrepancy detected later is compared against it to determine whether the issue is a migration error or a pre-existing source problem.

### Layer 2: Semantic-layer translation documentation

Every mapping from OBIEE constructs to Power BI equivalents is documented in `docs/rpd_to_dataflow_patterns.md` and `docs/data_dictionary.md`.

Known translation issues that change observable behavior are flagged explicitly:

- AR aging bucket timing: OBIEE calculated aging at query time using a session variable for the current date. Power BI calculates at refresh time. Documented in the data dictionary. Business owner signed off.
- Implicit aggregation: OBIEE allowed double-counting through alias table joins. Converted to explicit Snowflake view joins with deduplication logic. Net effect: some aggregate totals changed intentionally.
- Init block session variables: OBIEE used session variables populated at login. Power BI RLS uses USERPRINCIPALNAME() at query time. Behavior is equivalent for user-level filtering; org-hierarchy-based filtering required additional DAX logic.

### Layer 3: Automated validation suite

The validation suite runs after every conversion and before any report is marked as migrated.

**Record count checks** (`sql/validation/record_count_comparison.sql`)
Compares row counts between the OBIEE physical source and the Power BI dataset view. Threshold: within 1% variance. Any report outside threshold fails and requires root-cause documentation before proceeding.

**Aggregate total checks**
Compares sum, average, min, and max on all numeric measures between systems. Same 1% threshold. A report can pass record counts but fail aggregates if a calculation change affected measure values.

**Dimension value checks**
Compares distinct value sets on all dimension columns. Must be identical. Any dimension mismatch surfaces as a separate issue from count or aggregate failures, because the root cause and resolution path are different.

**Security audit** (`sql/validation/security_audit.sql`)
For each RLS role, runs the equivalent filter in both OBIEE and Power BI and compares row counts. Must match exactly. There is no tolerance for security discrepancies.

**Validation runner** (`scripts/run_validation.py`)
Automates the full suite against Snowflake. Loads thresholds from `config/validation_thresholds.yaml`. Returns structured pass/fail results per report per check type. Generates the validation report in `docs/validation_report.md`.

### Layer 4: Failure classification and triage

Validation failures are classified before resolution work begins. The classification determines the resolution path.

| Classification | Definition | Resolution |
|---|---|---|
| Migration error | The conversion logic introduced a discrepancy not present in the source | Fix the conversion and re-validate |
| Source data issue | The OBIEE source contained a defect that the migration faithfully reproduced | Document, do not replicate the defect, business owner accepts the correction |
| Intentional change | A known translation behavior change that was approved pre-migration | Document in data dictionary, confirm business owner sign-off |
| Threshold exception | Variance is within an acceptable range with a known cause | Document the cause and accept with owner approval |

In the migration documented in this repo: 237 of 240 reports passed on first run. The 3 failures were all classified as source data issues (reversed entries counted in OBIEE but excluded by the conversion logic, template placeholder entries included in OBIEE row counts, hardcoded date filters in OBIEE catalog not represented in Power BI view logic). All three were documented and accepted by business owners. None were re-introduced.

### Layer 5: Data Quality Issue Board (ongoing)

Post-migration, data quality issues are tracked through the Data Quality Issue Board in the Monday.com operating model.

- Detection sources: automated validation runs, business user reports, scheduled data audits, access reviews
- Each issue is assigned an owner, severity, and resolution deadline
- Issues do not close without evidence: a passing validation run or a written acceptance of a known discrepancy with business owner sign-off
- Chronic issues (same root-cause category recurring across multiple reports or refresh cycles) trigger a structural review

### Layer 6: Variance thresholds as configuration

`config/validation_thresholds.yaml` defines acceptable variance by check type:

```yaml
variance_tolerance_pct: 1.0      # Record counts and aggregates
dimension_match: exact            # Dimension values: no tolerance
security_match: exact             # RLS row counts: no tolerance
max_stale_hours: 24               # Data freshness threshold
```

Thresholds are intentionally strict. Raising a threshold to make a failure pass is not a valid resolution. Thresholds can only be changed through the change approval process with documented justification.

## Data dictionary as the source of truth

`docs/data_dictionary.md` is the living reference for all field definitions, business rule translations, and accepted discrepancies. It is updated:

- When a new field is added to a dataset view
- When a business rule translation changes
- When a known discrepancy is accepted by a business owner
- When a validation failure is resolved and the resolution changes field behavior

The data dictionary is the answer to the question: "Why does this number look different from what I saw in OBIEE?" If the answer is not in the data dictionary, the discrepancy has not been properly governed.

## Anti-patterns this framework prevents

- Closing a validation failure by adjusting the threshold to make the number fit
- Marking a report as migrated without a passing validation run
- Documenting a discrepancy in a ticket or email rather than the data dictionary
- Granting a security exception without updating the RLS mapping documentation
- Deploying a calculation change without a change approval record
