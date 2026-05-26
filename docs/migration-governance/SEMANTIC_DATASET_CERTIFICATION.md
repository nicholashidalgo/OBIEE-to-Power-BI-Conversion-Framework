# Semantic Dataset Certification

## 1. Purpose

This document defines what makes a semantic dataset certified for production use, the criteria that must be met before certification is granted, and the controls that govern certification maintenance over time. A certified dataset has documented grain, defined metric ownership, validated lineage, confirmed security controls, and a scheduled refresh that keeps data within the required freshness window.

Certification is not a one-time stamp. It is an ongoing status that can be revoked by an unresolved data quality issue, a failed refresh audit, a security change without documentation, or an RLS role change without re-validation.

---

## 2. Business Use Case

Users make decisions based on report data. When the semantic layer behind a report is undocumented—when no one knows which table a measure comes from, whether the join is correct, or why a particular filter was applied—the only recourse when a number looks wrong is to investigate from scratch. This is expensive, slow, and erodes confidence in the platform.

Dataset certification solves this by establishing explicit, documented conditions that a dataset must meet before it is trusted for business use. Certification also provides a contractual anchor: a certified dataset has a named owner who is accountable for its accuracy, a documented definition for every metric, and a clear process for making changes.

---

## 3. Users and Stakeholders

| Role | Responsibility |
|---|---|
| Dataset Owner | Accountable for the accuracy, documentation, and certification status of the dataset |
| Platform Engineer | Implements the dataset view, measures, and security; provides technical documentation |
| Security Reviewer | Validates RLS role coverage and confirms no unauthorized access |
| Data Steward | Maintains the data dictionary; reviews metric definitions for business accuracy |
| Business Stakeholder | Confirms that metric definitions match business intent |
| Platform Lead | Signs off on initial certification; reviews re-certification after material changes |

---

## 4. Migration Scope

This certification model was designed for the three datasets produced by this migration:

| Dataset | OBIEE Sources | Grain | RLS Roles |
|---|---|---|---|
| Finance | Finance - GL, Finance - AP, Finance - AR, Finance - Budget | Journal entry level; budget at account + cost center + month | Finance_NA, Finance_EMEA, Finance_APAC, Finance_Global |
| Risk | Risk - Credit, Risk - Operational | Risk event level | Risk_ReadOnly |
| Operations | Operations - Inventory, Operations - Shipping, Operations - Production, Operations - Quality | Transaction / operation record level | Operations_Plant |

Subject area consolidation rationale and grain decisions are documented in [`docs/subject_area_mapping.md`](../subject_area_mapping.md).

---

## 5. Operating Model

Dataset certification operates across three phases:

**Initial certification:** Executed once before a dataset's reports are released to users. Requires completion of all certification criteria listed in Section 7.

**Maintenance:** Certification status is reviewed after any material change—schema change, new measure, RLS modification, upstream source change—and after each quarterly access audit.

**Re-certification:** Required when a material change affects grain, metric definitions, or security. Minor additive changes (new column that does not affect existing measures or grain) do not require full re-certification but do require data dictionary update and change approval board sign-off.

**Revocation:** Certification is revoked when a Critical data quality issue is opened and not resolved within the defined SLA, when a security audit finds an RLS gap, or when the dataset fails its scheduled refresh validation check. Revoked datasets are flagged in the platform status board and users are notified.

---

## 6. Architecture

### Dataset View Architecture

Each Power BI dataset is backed by a Snowflake view in the `reporting` schema. The view implements:

- All joins from the RPD physical layer (translated from OBIEE LTS join logic)
- Any filters that should apply globally to the dataset (e.g., `is_reversed = FALSE`, `is_template = FALSE`)
- Derived columns that were RPD logical layer calculations but are not suitable for DAX (e.g., rolling window calculations)

View definitions: [`sql/powerbi_target/dataset_views.sql`](../../sql/powerbi_target/dataset_views.sql)

The Power BI semantic layer sits above the Snowflake views. It contains:

- DAX measures for all business logic calculations (aggregations, fiscal year rollups, variance calculations)
- RLS roles with DAX filter expressions applied to dimension tables
- Explicit relationships between fact and dimension tables (no implicit aggregation)

### Data Dictionary as the Certification Record

The data dictionary in [`docs/data_dictionary.md`](../data_dictionary.md) is the authoritative record for:

- Column-level mapping from OBIEE to Power BI (renamed columns, type changes, source table changes)
- Business rule translations (DAX equivalents for RPD derived columns and aggregation rules)
- Known discrepancies accepted by business owners with sign-off documentation

The data dictionary is a certification prerequisite: a dataset cannot be certified if any column lacks a documented mapping or if any known discrepancy lacks an acceptance record.

### Metric Ownership Model

Each metric in a certified dataset has:

- **Business owner:** The person accountable for the business definition
- **Technical owner (dataset owner):** The person accountable for the implementation
- **Definition:** What the metric means in business terms
- **Calculation:** How it is computed (DAX measure, Snowflake view column, or direct column)
- **Grain:** The lowest level at which the metric is meaningful
- **Known behaviors:** Any edge cases, timing differences, or accepted discrepancies

Sample metric ownership entries from [`docs/data_dictionary.md`](../data_dictionary.md):

| Metric | Business Definition | Calculation | Owner |
|---|---|---|---|
| Net Amount | GL journal entry net (debit minus credit) | `debit_amount - credit_amount` (Snowflake column) | FP&A |
| AR Aging Bucket | Days outstanding bucketed into aging categories | DAX SWITCH on `days_outstanding` | AR Team |
| Budget vs Actual Variance | Actual spend minus budgeted spend | `[OPEX Actual] - [OPEX Budget]` (DAX measure) | FP&A |
| Quality Score 3M Avg | 3-month rolling average quality score | `quality_score_3m_avg` (Snowflake window function) | Quality |

---

## 7. Controls

### Certification Criteria

A dataset must meet all of the following criteria to be certified:

**Grain documentation**
- [ ] The dataset grain is explicitly stated: the level at which one row exists
- [ ] The grain is confirmed against the source view (row count per grain key is 1)
- [ ] Any aggregated views (summary-level fact tables) have their grain documented separately

**Metric ownership**
- [ ] Every measure in the dataset has a named business owner
- [ ] Every measure has a documented definition in the data dictionary
- [ ] Every DAX measure has a documented equivalent in the source system (OBIEE derived column, RPD aggregation rule, or new metric post-migration)

**Column definitions and lineage**
- [ ] Every column in the Snowflake view is documented in the data dictionary with source table, transformation logic, and any known behavioral differences from the OBIEE source
- [ ] Column renames are documented (OBIEE name → Power BI name)
- [ ] Derived columns (DAX calculated columns and Snowflake window functions) have their calculation logic documented

**Relationships**
- [ ] All relationships between fact and dimension tables are documented
- [ ] No ambiguous relationships exist (no inactive relationships without explicit USERELATIONSHIP() documentation)
- [ ] Fan trap risks (one fact joining to two dimensions sharing a bridge) are identified and mitigated in the Snowflake view or through explicit DAX

**Row-level security**
- [ ] All RLS roles are documented in [`docs/rls_mapping.md`](../rls_mapping.md) with OBIEE source group, Power BI role name, DAX expression, and the dimension table the filter applies to
- [ ] Every role has been validated against OBIEE source output with exact row count match
- [ ] Users are assigned to roles via Azure Active Directory groups (not individual user assignments) where possible
- [ ] RLS role assignment is audited quarterly

**Refresh schedule**
- [ ] Scheduled refresh is configured in Power BI Service
- [ ] Refresh window matches the Snowflake data load schedule
- [ ] Data freshness threshold is defined: datasets must not exceed 24 hours stale (`max_stale_hours: 24` in `config/validation_thresholds.yaml`)
- [ ] Refresh failure notifications are configured to alert the dataset owner within 1 hour for datasets marked Critical

**Validation**
- [ ] The full validation suite has been run and results are in `docs/validation_report.md`
- [ ] All failures have been classified and either resolved or formally accepted with business owner sign-off
- [ ] Post-migration validation was run against the production-equivalent view (not a development schema)

**Change control**
- [ ] A change approval process is defined for any post-certification change (documented in `monday/board_schema.md` Change Approval Board)
- [ ] Tier assignments are documented: cosmetic/additive = Tier 1; logic/calculation = Tier 2; RLS/security = Tier 3
- [ ] The rollback plan for the dataset is documented

**Documentation completeness**
- [ ] `docs/data_dictionary.md` is complete for all columns in this dataset
- [ ] Subject area consolidation rationale is documented in `docs/subject_area_mapping.md`
- [ ] RPD-to-Power-BI pattern translation is documented in `docs/rpd_to_dataflow_patterns.md`
- [ ] No undocumented columns, measures, or relationships exist in the published dataset

---

## 8. Validation Workflow

### Pre-Certification Validation

Before a dataset certification is granted, run:

```bash
# Confirm the dataset view is populated and within threshold
python scripts/run_validation.py --dataset finance

# Confirm validation results are complete and documented
# Review docs/validation_report.md for this dataset

# Confirm data dictionary is complete for this dataset's columns
# Review docs/data_dictionary.md for coverage gaps

# Confirm RLS roles are documented and validated
# Review docs/rls_mapping.md
```

The unit tests confirm the validation logic is functioning:

```bash
pytest tests/
```

### Certification Checklist Execution

The certification checklist in Section 7 is reviewed by the dataset owner and platform lead. Each criterion is marked complete with a reference to the evidence:

- Grain documented → reference `docs/subject_area_mapping.md` grain row for this dataset
- Metrics owned → reference `docs/data_dictionary.md` business owner column
- RLS validated → reference `docs/rls_mapping.md` testing results row and `docs/validation_report.md` RLS section
- Validation passed → reference `docs/validation_report.md` overall results table

### Post-Change Re-Validation

After any change requiring re-certification:

1. Run `scripts/run_validation.py` for the affected dataset
2. If RLS was changed: run `sql/validation/security_audit.sql` for all affected roles
3. Update `docs/data_dictionary.md` for any new or changed fields
4. Update `docs/rls_mapping.md` if any role was modified
5. Record the change in the Change Approval Board
6. Confirm certification criteria are still met

---

## 9. Release Workflow

**Certification gates the release.** No dataset reports are released to users until the dataset is certified.

```
Dataset build complete
        │
        ▼
Certification checklist review (dataset owner + platform lead)
        │
        ├── Criteria met → Certified
        │
        └── Criteria not met → Track gaps in Change Approval Board
                                Resolve gaps and re-review
                                    │
                                    ▼
                              Certified
        │
        ▼
Reports in dataset released for UAT
        │
        ▼
UAT complete, stakeholder sign-off
        │
        ▼
Reports go live in Power BI Service
```

---

## 10. Operational Metrics

| Metric | Description |
|---|---|
| Data dictionary coverage | % of dataset columns with complete documentation |
| RLS role validation coverage | % of RLS roles validated against source with exact row match |
| Certification criteria pass rate | % of certification criteria met at initial review |
| Days to certification from dataset build complete | Elapsed time to complete all criteria |
| Re-certification events per quarter | Count of material changes requiring re-certification |
| Revocation events per year | Count of certification revocations; target is 0 |
| Data freshness compliance | % of refresh cycles completing within 24-hour freshness window |

---

## 11. Workplace Application

This certification model transfers to any environment where a team is accountable for the accuracy and trustworthiness of shared data assets. It applies equally to:

- Power BI datasets backed by any data warehouse (Snowflake, Redshift, BigQuery, Azure Synapse)
- Looker LookML models with named Explores and measures
- dbt semantic layer models with documented metrics
- Tableau Published Data Sources with shared metric definitions

**The underlying requirements are platform-agnostic:**

Every shared semantic layer should have documented grain, named metric owners, validated security controls, a defined refresh schedule, and a change control process that requires documentation before deployment.

The specific artifacts (DAX, Snowflake SQL, RLS roles) are substitutable. The governance structure is not.

---

## 12. Limitations

- This certification model is a designed governance framework. There is no automated certification tooling in this repo that programmatically checks and records certification status. Certification is a manual checklist review process.
- The data dictionary in `docs/data_dictionary.md` covers the Finance dataset column mappings. Risk and Operations column mappings follow the same pattern but are not expanded in the sample documentation.
- The certification model does not prescribe a specific tool for recording certification status. In practice, this would be tracked in a system of record (the Migration Status Board, a data catalog, or a shared document). The repo documents the criteria, not the tracking tool.
- RLS role validation was performed at migration time. Post-migration, re-validation requires both Snowflake access and a mechanism to compare results against a reference. This repo provides the SQL patterns but not an automated post-migration regression runner.

---

## 13. What This Does Not Claim

- This framework does not claim that a data catalog tool (Alation, Collibra, Atlan, or similar) is deployed or integrated. Certification is documented in Markdown and YAML artifacts within this repo, not in a production catalog system.
- This does not claim that Power BI sensitivity labels, information protection policies, or Microsoft Purview integration are configured. Those are platform-level settings outside the scope of this repo.
- The certification criteria listed here do not represent a regulatory or compliance framework. They are internal governance standards appropriate for a BI platform, not a substitute for regulatory data governance requirements (SOX data controls, GDPR lineage documentation, etc.), which have additional requirements beyond what is documented here.

---

## 14. Extension Path

- **Catalog integration:** Export the data dictionary in a format compatible with a data catalog tool (JSON-LD, Alation API format, or OpenMetadata schema) to enable catalog-driven certification workflows
- **Automated certification scoring:** Build a script that reads `docs/data_dictionary.md`, `docs/rls_mapping.md`, and `docs/validation_report.md` and scores each certification criterion as met or not met, surfacing gaps programmatically
- **Recertification triggers:** Add monitoring to the validation runner that flags when a dataset view changes (via Snowflake information_schema diffing) and creates a recertification task
- **Metric glossary:** Extend the data dictionary into a structured metric registry (YAML or JSON) that can be queried by report builders to confirm a metric's definition before using it
- **dbt metrics layer integration:** If the data transformation layer migrates to dbt, the metric ownership model maps directly to dbt's `metrics:` block syntax, enabling version-controlled metric definitions with documented owners and lineage

---

## 15. Interview Talking Points

**On certification as ongoing status:**
> "Certification is not a stamp. It's a status. Any material change to the dataset—a new measure, an RLS modification, a schema change upstream—puts the dataset back in review. That discipline is what prevents a certified dataset from drifting over time into an uncertified state with undocumented behavior."

**On grain documentation:**
> "The grain question is the most important one to answer before anything else. If you don't know what level one row represents, you can't correctly interpret any number the report produces. Grain documentation is the foundation everything else builds on."

**On metric ownership:**
> "Every metric needs a name attached to it—someone who is accountable for what it means and whether it's right. Without a named owner, when a business user says 'this number looks wrong,' there's no one whose job it is to answer that. Ownership doesn't have to mean someone wrote the SQL. It means someone can be called on to confirm the definition."

**On change control for certified datasets:**
> "The three-tier approval model is grounded in the risk profile of the change. A cosmetic rename is Tier 1. Changing how a measure is calculated is Tier 2 because it affects every report that uses that measure. Touching an RLS role is Tier 3 because it changes what rows users can see. The tier structure exists to ensure the right reviewers are involved in the right changes, not to slow things down."

**On what certification is not:**
> "Certification is an internal governance standard. It doesn't mean the dataset meets SOX requirements or GDPR lineage requirements—those are separate frameworks. What certification does mean is that anyone inside the organization can look up what every column means, who owns every metric, and what the security model allows. That's the baseline for a trusted platform."
