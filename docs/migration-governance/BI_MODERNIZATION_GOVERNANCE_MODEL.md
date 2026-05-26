# BI Modernization Governance Model

## 1. Purpose

This document defines the governance model for a full-platform BI modernization: the progression from a legacy reporting environment to a governed semantic dataset architecture and modern delivery platform. It covers inventory, report rationalization, semantic dataset ownership, security architecture, validation controls, change governance, release management, user adoption, and steady-state operations.

The model is grounded in the artifacts of this repository, which implements these patterns across a 240-report OBIEE to Power BI migration. Where a section references a specific repo artifact, that artifact exists and is functional. Where a section describes operating model guidance that extends beyond the implemented code, it is labeled accordingly.

---

## 2. Business Use Case

Enterprise BI platforms accumulate technical debt over time: duplicate reports, undocumented security rules, semantic model workarounds, and inconsistent business logic spread across dozens of subject areas. A modernization project replaces the platform while preserving the business logic. Done poorly, it creates confusion, erodes trust in data, and generates a long tail of user support issues. Done well, it delivers a more maintainable architecture, documented data contracts, and a governance model that prevents the same debt from re-accumulating.

This framework addresses the governance layer of that transition. The technical migration work—extraction, mapping, conversion, and validation—is necessary but not sufficient. Governance determines whether the new platform stays trustworthy over time.

---

## 3. Users and Stakeholders

| Role | Responsibility |
|---|---|
| BI Platform Owner / Manager | Overall accountability for platform architecture, governance, and operations |
| Dataset Owner | Business-accountable for the accuracy and fitness of a specific dataset (Finance, Risk, Operations) |
| Platform Engineer | Executes migrations, builds dataset views, writes DAX measures, maintains validation suite |
| Security Reviewer | Reviews and approves all RLS role changes and access grants |
| Business Analyst (Consumer) | Uses reports; raises data quality issues; participates in UAT and sign-off |
| Platform Lead | Approves Tier 1 changes; escalation point for unresolved issues |
| Business Stakeholder | Signs off on stakeholder review before a report is marked migrated |

---

## 4. Migration Scope

This governance model was applied to a migration covering:

- **Source platform:** OBIEE 12c with 10 subject areas and an RPD semantic layer
- **Target platform:** Power BI Desktop and Power BI Service backed by Snowflake
- **Report inventory:** 240 reports across Finance, Risk, and Operations
- **Semantic consolidation:** 10 OBIEE subject areas consolidated to 6 Power BI datasets
- **Security model:** OBIEE catalog permissions and initialization blocks translated to Power BI RLS roles with DAX filters
- **Active user base:** 500+ active report consumers

The 10-to-6 subject area consolidation was driven by RPD architecture constraints that did not reflect genuine business domain separation. Finance - AP, Finance - AR, Operations - Shipping, and Operations - Quality existed because the RPD could not expose the same physical table through different join paths in a single subject area. The underlying data was already in shared physical tables. Power BI does not have this constraint. Consolidation rationale and column mapping are documented in [`docs/subject_area_mapping.md`](../subject_area_mapping.md).

---

## 5. Operating Model

The governance model operates across three phases.

### Phase 1: Migration Governance

Covers the period from inventory extraction through stakeholder sign-off and OBIEE decommission. Governed by the migration playbook in [`docs/migration_playbook.md`](../migration_playbook.md).

Key governance controls during migration:
- No report is marked `migrated` in [`config/migration_inventory.yaml`](../../config/migration_inventory.yaml) without a passing validation run
- Validation failures are classified before resolution work begins (see [`docs/DATA_INTEGRITY_CONTROLS.md`](../DATA_INTEGRITY_CONTROLS.md))
- Known discrepancies are accepted by a named business owner and documented in [`docs/data_dictionary.md`](../data_dictionary.md), not suppressed
- RLS role translations are validated against OBIEE output using cross-join SQL audits; security parity is confirmed before cutover
- A 30-day parallel run period maintains OBIEE in read-only mode as a fallback

### Phase 2: Release and Cutover

Covers the transition from validated state to production delivery. Documented in [`docs/migration-governance/RELEASE_MANAGEMENT_AND_UAT_MODEL.md`](RELEASE_MANAGEMENT_AND_UAT_MODEL.md).

Key governance controls during release:
- Business owner sign-off recorded before each report goes live
- Rollback plan defined before cutover: OBIEE read-only for 30 days, both systems reading from the same Snowflake warehouse
- Hypercare period defined with dedicated support coverage
- Release notes document what changed for each cohort of reports

### Phase 3: Steady-State Operations

Covers ongoing platform governance after the migration is complete. Governed by [`docs/OPERATING_MODEL.md`](../OPERATING_MODEL.md).

The operating model uses a five-board workflow structure (modeled in Monday.com) to govern:
- New report intake and prioritization
- Migration status tracking and leadership visibility
- Data quality issue tracking from detection through resolution
- Change approval for RLS modifications, dataset view changes, and measure updates
- Access management and operational health

**Operating model status:** The Monday.com board architecture is a designed workflow model, not a deployed production integration. The board schemas, automation rules, and workflow examples in `monday/` are defined with enough specificity to implement. See the README and [`docs/CAPABILITY_ALIGNMENT.md`](../CAPABILITY_ALIGNMENT.md) for scope boundaries.

---

## 6. Architecture

### Semantic Layer Architecture

```
OBIEE RPD (Physical → Business Model → Presentation)
         │
         ▼
Inventory extraction (sql/obiee_source/)
         │
         ▼
Subject area rationalization (docs/subject_area_mapping.md)
         │
         ▼
Snowflake reporting views (sql/powerbi_target/dataset_views.sql)
  ├── reporting.v_finance_dataset
  ├── reporting.v_risk_dataset
  └── reporting.v_operations_dataset
         │
         ▼
Power BI datasets (Import or DirectQuery mode)
  ├── Finance dataset  (GL, AP, AR, Budget)
  ├── Risk dataset     (Credit Risk, Operational Risk)
  └── Operations dataset (Inventory, Shipping, Production, Quality)
         │
         ▼
Power BI reports → Power BI Service workspace → End users
```

### Governance Layer (Modeled)

```
Report Intake Board         ← New requests from business users
Migration Status Board      ← Progress tracking, leadership visibility
Data Quality Issue Board    ← Issue triage and resolution
Change Approval Board       ← Gated deployment control
Platform Operations Board   ← Access, refresh, incidents, enhancements
```

### Validation Architecture

```
Source extraction → scripts/run_validation.py
                         │
                         ├── Record count checks (1% threshold)
                         ├── Aggregate total checks (1% threshold)
                         ├── Dimension value checks (exact match)
                         └── RLS security checks (exact row match)
                              │
                              ▼
                    docs/validation_report.md
                         │
                         ▼
                   Pass/Fail determination
                    (config/validation_thresholds.yaml)
```

---

## 7. Controls

### Implemented Controls (functional in this repo)

| Control | Artifact | Description |
|---|---|---|
| Migration inventory | `config/migration_inventory.yaml` | Tracks all 240 reports with status, owner, priority, and validation state |
| Validation thresholds | `config/validation_thresholds.yaml` | Defines 1% tolerance for counts/aggregates; exact match for dimensions and security |
| Automated validation runner | `scripts/run_validation.py` | Executes record count, aggregate, and dimension checks against Snowflake views |
| Validation report | `docs/validation_report.md` | Structured pass/fail output with root-cause documentation for all failures |
| Security audit SQL | `sql/validation/security_audit.sql` | Cross-join audit confirming identical row sets per RLS role |
| RLS mapping documentation | `docs/rls_mapping.md` | Full group-to-role mapping with DAX expressions and test results |
| Data dictionary | `docs/data_dictionary.md` | Column mappings, business rule translations, and accepted discrepancies |
| Unit tests | `tests/test_validation.py` | Pytest coverage for validation logic (passing: 3/3) |
| Migration status generator | `scripts/generate_mapping_report.py` | Generates status summary from inventory YAML |

### Operating Model Controls (designed, not deployed as production integrations)

| Control | Reference | Description |
|---|---|---|
| Intake review gate | `monday/board_schema.md` | No report enters build queue without completed intake review |
| Deployment gate | `monday/board_schema.md` | Change Approval Board holds deployment until all required approvals are recorded |
| Escalation automation | `monday/automation_rules.md` | High/Critical issues open 14+ days without a resolution plan escalate to platform leadership |
| Quarterly access audit | `docs/OPERATING_MODEL.md` | RLS role assignments reviewed quarterly against active user list |
| Data dictionary governance | `docs/OPERATING_MODEL.md` | Any change to a dataset view or measure definition requires a data dictionary update before deployment |

---

## 8. Validation Workflow

The validation workflow is executed per report, and then as a full-suite run before any cohort of reports is marked ready for release.

**Per-report sequence:**

1. Export OBIEE report output to CSV (manual step; requires OBIEE access)
2. Run Power BI dataset view query and export to CSV
3. Compare row counts: must be within 1% (`variance_tolerance_pct: 1.0` in `config/validation_thresholds.yaml`)
4. Compare aggregate totals on all numeric measures: must be within 1%
5. Compare distinct dimension values: must be identical (`dimension_match: exact`)
6. For secured reports: run as each RLS role and compare row counts in both systems: must match exactly (`security_match: exact`)

Steps 2–6 are automated by `scripts/run_validation.py`. Step 1 requires manual OBIEE access.

**On failure:**

1. Classify the failure using the four-category framework in `docs/DATA_INTEGRITY_CONTROLS.md`:
   - Migration error → fix conversion logic and re-validate
   - Source data issue → document, do not replicate, get business owner acceptance
   - Intentional change → confirm pre-approved, document in data dictionary
   - Threshold exception → document cause, get business owner acceptance
2. Record the root cause and resolution in the validation report
3. Update `docs/data_dictionary.md` if the resolution changes observable field behavior
4. Re-run the failing checks to confirm resolution before marking the report as migrated

**Results from this repo's migration (documented, not invented):**

- 237 of 240 reports passed all checks on first validation run
- 3 record count failures: all classified as source data issues (reversed entries, template placeholders, hardcoded date filters in OBIEE)
- 2 dimension value failures: new dimension members added to source system during migration window
- 0 security failures: all RLS roles confirmed with exact row count parity
- Full details in `docs/validation_report.md`

---

## 9. Release Workflow

See [`docs/migration-governance/RELEASE_MANAGEMENT_AND_UAT_MODEL.md`](RELEASE_MANAGEMENT_AND_UAT_MODEL.md) for the full release governance model.

**Summary sequence:**

1. Validation suite passes for all reports in the release cohort
2. UAT period: business users run migrated reports alongside OBIEE for a defined window
3. Discrepancies raised during UAT are triaged using the failure classification framework
4. Business owner sign-off recorded per report or per cohort
5. Release notes published: maps old OBIEE report names to new Power BI report names
6. Cutover executed: users directed to Power BI
7. OBIEE retained in read-only mode for 30 days (rollback window)
8. Hypercare period: elevated support coverage for the first two weeks post-cutover
9. OBIEE decommissioned after 30-day parallel run with no unresolved issues

**Rollback condition:** If a critical issue is found after cutover that cannot be resolved within the hypercare period, users are directed back to OBIEE for the affected reports while the issue is investigated. No data is lost because both systems read from the same Snowflake warehouse.

---

## 10. Operational Metrics

These are the metrics a BI platform owner monitors to evaluate platform health. Targets are reference architecture guidance; actual values depend on organizational context.

| Metric | Description | Reference Target |
|---|---|---|
| Validation pass rate | % of reports passing all validation checks on first run | ≥95% |
| First-run security parity | % of RLS roles with exact row count match | 100% |
| Data dictionary coverage | % of columns with documented mappings and business rules | 100% at release |
| Mean time to resolve data quality issue | Days from detection to closure | <7 days (High severity) |
| Open issues by severity | Count of Critical/High issues open at any time | 0 Critical, <3 High |
| Validation cycle time | Elapsed time from report conversion to validation pass | Tracked per cohort |
| Intake review turnaround | Days from submission to intake decision | ≤3 business days |
| Access audit coverage | % of RLS roles reviewed against current user list in last quarter | 100% |
| Refresh success rate | % of scheduled dataset refreshes completing without failure | ≥99% |

**Metrics from this repo's migration** (documented, derived from `docs/validation_report.md` and `config/migration_inventory.yaml`):

- Validation pass rate (first run): 98.8% record counts, 100% aggregates, 100% RLS
- Security parity: 100% (240/240 reports, exact row match per user group)
- All 3 record count failures and 2 dimension failures documented with root cause; none re-introduced

---

## 11. Workplace Application

This governance model is transferable to any enterprise BI platform modernization. The specific implementations—OBIEE to Power BI, Snowflake views as the semantic layer, Monday.com as the workflow layer—are substitutable. The governance structure applies to:

- Tableau to Power BI migrations
- Cognos, MicroStrategy, or Business Objects to any modern BI platform
- Data warehouse re-platforming where downstream reports must maintain parity
- Platform consolidations where multiple BI tools are unified under a single governed environment

**What transfers directly:**

- The validation control structure (inventory → translate → validate → document → release)
- The failure classification framework (migration error, source data issue, intentional change, threshold exception)
- The data dictionary as the living record of all field behavior changes
- The change approval tier structure for governing post-migration platform changes
- The access management model with quarterly audit requirements
- The escalation paths and resolution accountability model

**What must be adapted:**

- Specific SQL and DAX patterns depend on the source/target platform combination
- Threshold values may vary by organization based on data volume and acceptable risk
- Workflow tooling (Monday.com, Jira, ServiceNow, or equivalent) must match the organization's existing toolchain

---

## 12. Limitations

- The Monday.com operating layer is a designed workflow architecture, not a deployed production integration. The board schemas, automation rules, and GraphQL examples demonstrate design capability and are implementable, not running in a live tenant.
- The validation runner (`scripts/run_validation.py`) requires Snowflake connectivity. Running the full suite against live data requires configuring Snowflake credentials in `.env`. The unit tests in `tests/test_validation.py` run without a live connection.
- Sample data in `data/sample/` is illustrative. The 10 reports shown in the sample represent the structural pattern for the full 240-report inventory. The full inventory is not included because it contains information specific to the original engagement.
- `config/migration_inventory.yaml` contains 10 sample entries; the comment at line 84 notes the pattern extends to all 240 reports. Actual migration outcomes referenced in `docs/validation_report.md` and `docs/EXECUTIVE_SUMMARY.md` are the real results from that migration.
- This framework does not include Power BI workspace configuration, PBIX files, or Snowflake DDL for the reporting views. Those contain organization-specific business logic.

---

## 13. What This Does Not Claim

- This is not a deployed production Power BI tenant. No live Power BI Service workspace is part of this repo.
- This does not claim Monday.com automation rules are executing against a real board. The automation logic is designed and documented, not running.
- This does not claim the 240-report validation results are reproducible from this repo alone. They required OBIEE access and Snowflake access that is not part of this public repository.
- This does not claim specific cost savings, productivity gains, or adoption percentages. No such figures are invented or implied.
- The governance model does not claim to be the only valid approach to BI platform governance. It is a documented pattern based on a real migration, not a universal standard.

---

## 14. Extension Path

The following extensions are within the scope of this framework's architecture and are documented as natural next steps, not current capabilities:

- **Automated data quality alerting:** Integrate `scripts/run_validation.py` with a notification layer (email, Slack, or Monday.com webhook) to surface failures without manual report review
- **Certification registry:** Extend `config/migration_inventory.yaml` with certification metadata fields (certified_by, certified_date, recertification_due) to support a formal dataset certification process (see [`docs/migration-governance/SEMANTIC_DATASET_CERTIFICATION.md`](SEMANTIC_DATASET_CERTIFICATION.md))
- **Incremental validation:** Adapt the validation runner to support delta validation on new data loaded since the last run, not just full-table comparisons
- **Cross-platform parity:** Extend the validation pattern to compare two live systems (Power BI vs. Snowflake query) for ongoing regression monitoring, not just initial migration parity
- **Lineage tracking:** Integrate column-level lineage (source table → Snowflake view → Power BI column → DAX measure) into the data dictionary using a structured format compatible with data catalog tooling

---

## 15. Interview Talking Points

These talking points summarize what this governance model demonstrates in workplace terms. They are accurate to the repo and should be discussed by reference to the artifacts, not claimed as abstract expertise.

**On platform governance:**
> "I designed the governance model around a separation of concerns: the technical migration controls are in code and configuration, and the operational governance is documented as an operating model. The data dictionary, validation thresholds, and migration inventory are the source of truth for what was done and why. Every accepted discrepancy has a business owner's name attached to it."

**On validation controls:**
> "The framework has a four-category failure classification—migration error, source data issue, intentional change, threshold exception—because how you classify the failure determines how you resolve it. A migration error means fix the code. A source data issue means document it and confirm the new system is actually correct. Those two require completely different responses, and collapsing them into a generic 'variance' bucket leads to bad decisions."

**On security parity:**
> "RLS validation is not optional. For every role, I ran the same query against both systems and compared row counts. The threshold for security checks is exact match—zero tolerance. You cannot accept a 1% variance on who can see which rows. The cross-join SQL audit approach in this repo ensures no role is marked valid until the row sets are identical."

**On data dictionary governance:**
> "The data dictionary is not documentation you write once and forget. Every time a calculation changes, every time a discrepancy is accepted, the dictionary gets updated before the change is deployed. That discipline is what prevents the new platform from accumulating the same undocumented technical debt as the old one."

**On the operating model:**
> "The Monday.com layer is designed to make the platform governable at scale. Without a structured intake process, you end up building reports that don't have business owners and can't be rationalized later. Without a change approval gate, security changes get made verbally and nobody can reconstruct what happened during an audit."
