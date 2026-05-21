# Enterprise Business Systems Ownership

This document maps the capabilities demonstrated in this repository to the functional areas of enterprise business systems ownership. It is a reference for understanding what this framework proves and what it is designed to show.

## Capability map

### Platform architecture

**What this repo shows:**
Redesigned the reporting semantic layer by replacing the OBIEE RPD with Snowflake views and a Power BI dataset architecture. Consolidated 10 subject areas into 6 datasets. Documented consolidation rationale, join patterns, and the trade-offs between RPD constructs and Power BI equivalents.

**Artifacts:** `docs/rpd_to_dataflow_patterns.md`, `docs/subject_area_mapping.md`, `sql/powerbi_target/`

---

### Workflow design

**What this repo shows:**
Modeled a five-board Monday.com operating layer that governs the full lifecycle of a BI platform: report intake, migration status tracking, data-quality issue triage, change approvals, and steady-state operations. Board schemas, automation rules, and workflow examples are defined with enough specificity to implement.

**Artifacts:** `monday/board_schema.md`, `monday/workflow_examples.md`, `monday/automation_rules.md`

---

### Operational reporting

**What this repo shows:**
Built an automated Python validation runner that executes the full suite of record count, aggregate, dimension, and security checks against Snowflake. Generates a structured validation report used as the go/no-go gate for each report. Migration status is tracked programmatically via `config/migration_inventory.yaml` and surfaced through `scripts/generate_mapping_report.py`.

**Artifacts:** `scripts/run_validation.py`, `scripts/generate_mapping_report.py`, `docs/validation_report.md`

---

### Integrations

**What this repo shows:**
Modeled Monday.com GraphQL integration patterns for reading board state, writing item updates, and triggering automation from external systems. Sample payloads cover the full event surface: intake created, status changed, approval required, data-quality issue opened, deployment approved.

**Artifacts:** `monday/graphql_examples.md`, `monday/sample_payloads/`

---

### Intake and prioritization

**What this repo shows:**
Designed a Report Intake Board that captures business owner, source system, dataset target, priority justification, and expected user count at submission. Intake review gates entry into the build queue. Priority scoring and backlog ordering are defined in the board schema.

**Artifacts:** `monday/board_schema.md` (Report Intake Board), `monday/workflow_examples.md`

---

### Data integrity

**What this repo shows:**
Defined variance thresholds in configuration. Ran automated validation across 240 reports covering record counts, aggregates, dimension values, and row-level security. Documented every known discrepancy with root-cause analysis. Built a Data Quality Issue Board that tracks each issue from detection through resolution with owner accountability.

**Artifacts:** `docs/DATA_INTEGRITY_CONTROLS.md`, `docs/validation_report.md`, `config/validation_thresholds.yaml`, `sql/validation/`

---

### Access controls

**What this repo shows:**
Extracted every row-level security rule from the OBIEE catalog. Mapped OBIEE groups to Power BI RLS roles. Wrote and validated DAX filter expressions. Confirmed exact row-count parity for all user groups using cross-join SQL audits. Documented the full group-to-role mapping with DAX expressions.

**Artifacts:** `docs/rls_mapping.md`, `sql/powerbi_target/rls_implementation.sql`, `sql/validation/security_audit.sql`

---

### Scalable systems architecture

**What this repo shows:**
The framework is designed to be generalized. Validation thresholds are configuration-driven. The migration inventory is YAML-structured and tool-readable. The Monday.com board schemas define a repeatable operating model that scales to additional platforms, teams, and report portfolios.

**Artifacts:** `config/`, `docs/OPERATING_MODEL.md`, `monday/`

---

### Leadership visibility

**What this repo shows:**
Designed the Migration Status Board as the leadership-facing view: report counts by phase, dataset rollup, blocked items, and sign-off status. The Executive Summary provides the outcome narrative. The operating model defines what leadership sees at steady state.

**Artifacts:** `docs/EXECUTIVE_SUMMARY.md`, `monday/board_schema.md` (Migration Status Board), `monday/workflow_examples.md`
