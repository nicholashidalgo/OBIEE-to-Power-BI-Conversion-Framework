# Report Reconciliation and Parity Testing

## 1. Purpose

This document defines the reconciliation and parity testing methodology used to confirm that migrated reports in the target platform produce results that match the source system to within defined tolerances. It covers check types, threshold definitions, failure classification, exception handling, security parity testing, edge cases, and the sign-off process.

Parity testing is the gate between a completed migration and a report being marked production-ready. A report that has been converted but not reconciled is not migrated—it is a draft.

---

## 2. Business Use Case

When an organization migrates reporting from one platform to another, users need confidence that the numbers they see on the new platform are correct. "Correct" in this context means consistent with the source system—adjusted only where the source system was wrong, and documented in all such cases.

Without a structured reconciliation process, discrepancies surface after go-live through user reports. Each unplanned discrepancy erodes trust and generates support volume. A structured parity testing process finds and resolves discrepancies pre-launch, classifies them with business owner acceptance, and gives the platform team a defensible record of what changed and why.

---

## 3. Users and Stakeholders

| Role | Responsibility in Parity Testing |
|---|---|
| Platform Engineer | Executes validation checks; investigates failures; proposes resolution classification |
| Dataset Owner | Reviews and accepts discrepancies classified as source data issues or intentional changes |
| Business Owner (per report) | Signs off on reconciliation results before a report is marked production-ready |
| Security Reviewer | Signs off on RLS parity results for secured reports |
| Platform Lead | Approves threshold exceptions and escalated failures |

---

## 4. Migration Scope

The reconciliation framework in this repo was applied to:

- **240 reports** across Finance, Risk, and Operations subject areas
- **6 Power BI datasets** (consolidated from 10 OBIEE subject areas)
- **6 RLS roles** with DAX filter expressions replacing OBIEE catalog security groups

Sample report inventory structure: [`data/sample/sample_report_inventory.csv`](../../data/sample/sample_report_inventory.csv)
Full inventory with validation status: [`config/migration_inventory.yaml`](../../config/migration_inventory.yaml)

---

## 5. Operating Model

Parity testing operates in two modes:

**Pre-release testing (migration gate):** Executed for every report before it is marked `migrated` in the inventory. Required for go-live approval.

**Post-release regression testing (ongoing governance):** Re-executed on a scheduled basis or after any change to a dataset view, measure definition, or RLS role. Governed by the Change Approval Board model in [`docs/OPERATING_MODEL.md`](../OPERATING_MODEL.md).

Parity testing is not a one-time activity. Any change to the semantic layer—a new Snowflake view column, a DAX measure edit, a schema change upstream—can introduce a regression. The automated validation runner is designed to be re-run after any such change.

---

## 6. Architecture

### Check Types and Their Purpose

| Check Type | What It Validates | Threshold | Artifact |
|---|---|---|---|
| Record count | Total row count in Power BI view vs. source | Within 1% | `sql/validation/record_count_comparison.sql` |
| Aggregate total | Sum, average, min, max on numeric measures | Within 1% | `scripts/run_validation.py` |
| Dimension value | Distinct values on all dimension columns | Exact match | `scripts/run_validation.py` |
| RLS security | Row count per user role vs. source system row count | Exact match | `sql/validation/security_audit.sql` |
| Filter behavior | Report-level filter combinations produce equivalent results | Exact match for security; 1% for aggregates | Manual/SQL |
| Drill path | Hierarchy navigation returns equivalent members | Exact match | Manual |
| Edge case | Null values, zero-amount rows, reversed entries, template rows | Documented | Manual |

### Threshold Configuration

Thresholds are defined in [`config/validation_thresholds.yaml`](../../config/validation_thresholds.yaml):

```yaml
variance_tolerance_pct: 1.0   # Record counts and aggregates: within 1%
dimension_match: exact         # Dimension distinct values: no tolerance
security_match: exact          # RLS row counts: no tolerance
max_stale_hours: 24            # Data freshness: data must be refreshed within 24 hours
```

**Why 1% for counts and aggregates:** The 1% threshold accommodates timing differences between systems (both reading the same Snowflake warehouse but potentially at different refresh points), rounding differences in derived calculations, and known source data behaviors where the source system included data that should be excluded. It does not accommodate migration logic errors. Any failure within the 1% band still requires classification and documentation.

**Why exact match for dimensions and security:** Dimension mismatches cannot be averaged away—they indicate either missing or extra members, which changes report filtering options for users. Security mismatches cannot be tolerated at any level: a user who can see one extra row they are not entitled to see is a security failure, regardless of the percentage.

### Automated Validation Runner

`scripts/run_validation.py` automates the record count, aggregate, and dimension checks against Snowflake. It:

- Loads thresholds from `config/validation_thresholds.yaml`
- Accepts a `--dataset` argument to run against a single dataset (`finance`, `risk`, or `operations`)
- Returns structured pass/fail results per dataset per check type
- Logs all results to standard output

Running the full suite:
```bash
python scripts/run_validation.py
```

Running against one dataset:
```bash
python scripts/run_validation.py --dataset finance
```

The runner requires Snowflake credentials in `.env` (see `.env.example`). Unit tests in `tests/test_validation.py` verify the validation logic without a live connection.

---

## 7. Controls

### Pre-Release Gate

A report cannot be marked `migrated` in `config/migration_inventory.yaml` until:

1. Record count check passes (within 1%) or failure is documented and accepted
2. Aggregate total check passes (within 1%) or failure is documented and accepted
3. Dimension value check passes (exact match) or failure is documented with root cause
4. RLS security check passes (exact match) for all applicable roles
5. All accepted discrepancies are recorded in `docs/data_dictionary.md` with business owner sign-off
6. Business owner sign-off is recorded in the Migration Status Board

This gate is enforced by process, not by code lock. The inventory YAML is the record.

### Failure Classification

All failures are classified before resolution work begins. The classification determines the resolution path.

| Classification | Definition | Resolution Path |
|---|---|---|
| Migration error | The conversion logic introduced a discrepancy not present in the source | Fix the conversion logic and re-validate |
| Source data issue | The source system contained a defect that the migration faithfully reproduced | Document the defect; confirm Power BI behavior is correct; business owner accepts |
| Intentional change | A known translation behavior change approved before migration began | Confirm pre-approval; document in data dictionary; no further action |
| Threshold exception | Variance is within the acceptable range but has a known, documented cause | Document cause; business owner accepts |

**Raising thresholds to pass a failure is not a valid resolution.** Thresholds may only be changed through the Change Approval Board with documented justification.

### Anti-Patterns This Framework Prevents

- Marking a report migrated before the validation suite has been run
- Closing a security failure with a comment rather than a confirmed row count match
- Accepting a failure by adjusting the threshold rather than investigating the root cause
- Documenting a discrepancy in a ticket or email instead of the data dictionary
- Signing off on reconciliation results without a named business owner

---

## 8. Validation Workflow

### Standard Sequence Per Report

**Step 1: Run the OBIEE report**
Export the OBIEE report output to CSV. This step requires OBIEE access and must be performed by a team member with appropriate catalog permissions. The export captures the exact row set the user sees, including any implicit filters applied by the OBIEE presentation layer.

**Step 2: Run the Power BI dataset view query**
Execute the equivalent Power BI query against the Snowflake reporting view. The `scripts/run_validation.py` runner automates this step.

**Step 3: Record count comparison**
Compare total row counts. If variance exceeds 1%, the report fails the count check. Record the OBIEE count, the Power BI count, the variance percentage, and the initial hypothesis for the cause.

**Step 4: Aggregate total comparison**
Compare SUM, AVG, MIN, MAX on all numeric measure columns. Apply the 1% threshold per measure. A report can pass record counts but fail aggregates—these are independent checks.

**Step 5: Dimension value comparison**
Compare distinct value sets on all dimension columns. Any missing or extra value is a failure regardless of count. Record which values differ.

**Step 6: RLS security comparison (secured reports only)**
For each RLS role that applies to the report, execute the equivalent filter in both systems and compare row counts. Row counts must match exactly. Refer to `sql/validation/security_audit.sql` for the cross-join audit pattern and [`docs/rls_mapping.md`](../rls_mapping.md) for the role-to-filter mapping.

**Step 7: Classify and document any failures**
Apply the failure classification framework. Update `docs/validation_report.md` with the results. Update `docs/data_dictionary.md` for any accepted discrepancy.

**Step 8: Get business owner sign-off**
The business owner reviews the reconciliation results, including any accepted discrepancies. Sign-off is recorded before the report is marked migrated.

### Edge Cases Requiring Explicit Handling

| Edge Case | Handling Approach |
|---|---|
| Reversed entries | OBIEE included reversed entries in counts; Power BI view filters them (`is_reversed = FALSE`). Document as source data issue; Finance confirmed Power BI is correct. |
| Template placeholder rows | OBIEE included template entries (`is_template = TRUE`); Power BI view excludes them. Document as source data issue. |
| Hardcoded date filters in OBIEE | Some OBIEE reports had hardcoded date range filters that were not exposed to users. Power BI equivalents use dynamic slicers. Validate by applying the equivalent filter and comparing. |
| New dimension members added during migration | Source system added new members after OBIEE extract but before Power BI view was created. Refresh dimension tables and re-validate. |
| Null values on measure columns | Confirm null handling is consistent. OBIEE RPD may have had default-zero behavior that Power BI does not replicate by default. |
| Fiscal year boundaries | Validate aggregate rollups at fiscal year cutoff. This framework uses a June 30 fiscal year end in DAX `DATESYTD`. |
| Multi-role users | A user assigned to multiple RLS roles should see the union of rows permitted by each role. Confirm this behavior is consistent with OBIEE group membership behavior. |

---

## 9. Release Workflow

Parity testing feeds directly into the release workflow defined in [`docs/migration-governance/RELEASE_MANAGEMENT_AND_UAT_MODEL.md`](RELEASE_MANAGEMENT_AND_UAT_MODEL.md).

**Milestone: All reports in cohort pass parity testing**
→ Cohort moves to UAT phase

**Milestone: UAT discrepancies triaged and resolved**
→ Business owner sign-off collected per report or per cohort

**Milestone: Sign-off complete, release notes published**
→ Cutover authorized

**Post-cutover validation:**
Re-run `scripts/run_validation.py` against the live production view after the first scheduled data refresh. Confirm row counts and aggregates are still within threshold. Any regression after cutover creates a Critical data quality issue in the Data Quality Issue Board.

---

## 10. Operational Metrics

| Metric | Description |
|---|---|
| Parity pass rate (first run) | % of reports passing all checks on the first validation run |
| Security parity rate | % of RLS roles with exact row count match |
| Mean time to classify failure | Time from failure detection to classification |
| Mean time to resolve failure | Time from classification to re-validation pass |
| Open accepted discrepancies | Count of discrepancies documented and accepted; reviewed quarterly |

**Results from this repo's migration** (from `docs/validation_report.md`):

| Check | Reports | Passed | Failed | Pass Rate |
|---|---|---|---|---|
| Record count (within 1%) | 240 | 237 | 3 | 98.8% |
| Aggregate totals (within 1%) | 240 | 240 | 0 | 100% |
| Dimension values (exact match) | 240 | 238 | 2 | 99.2% |
| RLS security (exact row match) | 240 | 240 | 0 | 100% |

All 3 record count failures and 2 dimension failures were classified, documented, and accepted by business owners. None were resolved by threshold adjustment. Details in [`docs/validation_report.md`](../validation_report.md).

---

## 11. Workplace Application

This parity testing framework is applicable to any migration where a legacy reporting system is being replaced and business users need documented assurance that the new system produces equivalent results.

**Common adaptations:**

- Replace OBIEE SQL extract steps with the equivalent for the source system (Tableau REST API, Cognos report export, Business Objects query)
- Replace Snowflake connectivity in `scripts/run_validation.py` with the target data warehouse connector
- Adjust thresholds in `config/validation_thresholds.yaml` to reflect organizational risk tolerance
- Adapt the RLS security check for the target platform's access control model (Tableau row-level security, Looker access grants, etc.)

**What remains constant across contexts:**

- The four-category failure classification framework
- The requirement for business owner sign-off before a report is marked production-ready
- The data dictionary as the record of all accepted discrepancies
- The zero-tolerance policy for security parity failures

---

## 12. Limitations

- The automated validation runner requires a live Snowflake connection and appropriate credentials. The unit tests in `tests/test_validation.py` cover the logic without a live connection.
- Step 1 (OBIEE export) is manual. Full automation of the end-to-end comparison requires either OBIEE API access or an agent that can execute OBIEE queries programmatically. This framework does not include that automation.
- Filter behavior and drill path checks are currently manual. The SQL validation patterns cover row counts, aggregates, and dimension values. Filter and drill-path checks require a human tester with access to both systems.
- The validation runner compares at the dataset level (full table), not at the individual report level. Report-level filter combinations are validated through manual test steps and UAT.

---

## 13. What This Does Not Claim

- The validation results in `docs/validation_report.md` are documented outcomes from the actual migration. They are not reproducible from this public repo alone, which lacks OBIEE access and live Snowflake credentials.
- This framework does not claim to detect all possible discrepancy types. Edge cases not covered in this document may exist and should be identified through UAT.
- The 98.8% first-run pass rate is specific to this migration. It reflects the quality of the source system and the rigor of the pre-migration inventory. It is not a guaranteed outcome for other migrations.
- This framework does not validate report formatting, visualization behavior, calculated KPI display, or conditional formatting—only data values and security row sets.

---

## 14. Extension Path

- **Report-level validation:** Extend the runner to validate at the individual report level (using specific filter conditions per report) rather than only at the dataset view level
- **Continuous regression testing:** Schedule the validation runner to execute after every dataset refresh and alert on any threshold breach
- **Delta validation:** Validate only rows added or changed since the last validation run, reducing computational cost on large datasets
- **Side-by-side comparison output:** Generate a structured CSV or HTML diff output showing exactly which row counts, aggregates, or dimension values differ, rather than just pass/fail counts
- **Integration with Data Quality Issue Board:** Have the validation runner automatically create issue board items (via Monday.com GraphQL) for any failing check, eliminating the manual issue creation step

---

## 15. Interview Talking Points

**On the validation design:**
> "The framework separates four independent check types—row counts, aggregates, dimension values, and security—because each has a different failure mode and resolution path. A row count failure with passing aggregates tells you something different from an aggregate failure with passing row counts. Collapsing them into a single 'does it match' check loses that diagnostic signal."

**On the 1% threshold:**
> "The 1% threshold is not a get-out-of-jail card. Any failure within the 1% band still has to be classified and documented. The threshold exists because both systems are reading from Snowflake but not necessarily at the same refresh point, and some source data behaviors produce systematic differences that are intentional. What the threshold does not do is let you ignore a failure you don't understand."

**On security parity:**
> "Security checks have zero tolerance. I ran the equivalent filter for every RLS role against both OBIEE and Power BI and compared row counts. If they don't match exactly, the role is not validated. There's no percentage variance acceptable for access control. Either a user sees the correct rows or they don't."

**On failure classification:**
> "Before you can resolve a failure, you have to classify it. A migration error and a source data issue look similar—both show a variance—but they require opposite responses. A migration error means your conversion logic is wrong and you need to fix it. A source data issue often means Power BI is actually more correct than OBIEE was, and your job is to document why and get the business owner to confirm."

**On documentation discipline:**
> "Every accepted discrepancy goes into the data dictionary with the business owner's acknowledgment. That record is what lets the support team answer the question 'why does this number look different' six months after go-live, without having to re-investigate from scratch."
