<div align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/nicholasjh-work/OBIEE-to-Power-BI-Conversion-Framework/main/assets/nh-logo-dark.svg">
    <img alt="NH" src="https://raw.githubusercontent.com/nicholasjh-work/OBIEE-to-Power-BI-Conversion-Framework/main/assets/nh-logo-light.svg" height="48">
  </picture>
</div>

<h1 align="center">Enterprise BI Platform Migration and Operations Framework</h1>

<p align="center">
  <strong>Production-pattern enterprise BI migration framework for governed OBIEE to Power BI modernization</strong>
</p>

<p align="center">
  <a href="http://obiee.nicholashidalgo.com">
    <img src="https://img.shields.io/badge/Demo-Live%20Dashboard-0078D4?style=for-the-badge&logo=powerbi&logoColor=white" alt="Live Dashboard">
  </a>
  <a href="LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-22c55e?style=for-the-badge" alt="License: MIT">
  </a>
  <a href="scripts/run_validation.py">
    <img src="https://img.shields.io/badge/Validation-Framework-f59e0b?style=for-the-badge" alt="Validation Framework">
  </a>
  <a href="monday/">
    <img src="https://img.shields.io/badge/Components-5-8b5cf6?style=for-the-badge" alt="Components: 5">
  </a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/SQL-4479A1?style=flat&logo=postgresql&logoColor=white" alt="SQL">
  <img src="https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black" alt="Power BI">
  <img src="https://img.shields.io/badge/Oracle%20OBIEE-F80000?style=flat&logo=oracle&logoColor=white" alt="Oracle OBIEE">
  <img src="https://img.shields.io/badge/Monday.com%20(modeled)-F62B54?style=flat&logo=monday&logoColor=white" alt="Monday.com modeled">
  <img src="https://img.shields.io/badge/Snowflake-29B5E8?style=flat&logo=snowflake&logoColor=white" alt="Snowflake">
</p>

---

<table width="100%">
<tr>
<td>

**What This Does**

This repository models an enterprise BI platform migration from Oracle Business Intelligence Enterprise Edition (OBIEE) to Power BI across 240 reports, 10 subject areas, and 6 Power BI datasets. It covers migration inventory, semantic-layer mapping, security mapping, validation SQL, a Python validation runner, and reporting continuity controls. The framework provides a reusable, auditable pattern for governing a large-scale BI platform conversion from initial inventory through stakeholder sign-off and decommission.

The framework documents release-readiness checks, data dictionary management, validation thresholds, access-control mapping, data-integrity controls, and a modeled Monday.com operating layer for intake, prioritization, change approval, issue triage, and stakeholder visibility. The Monday.com layer is a modeled workflow architecture, not a deployed production integration. It demonstrates how a business-facing orchestration platform governs a BI system across its full lifecycle: migration, validation, change control, and steady-state operations.

</td>
</tr>
</table>

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│         OBIEE Reports + RPD Semantic Layer        │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│          Migration Inventory and Mapping          │
│   config/migration_inventory.yaml  (240 reports) │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│              Semantic Translation                 │
│  docs/rpd_to_dataflow_patterns.md                │
│  docs/subject_area_mapping.md                    │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│            Power BI Dataset Design                │
│  sql/powerbi_target/dataset_views.sql            │
│  powerquery/common_transforms.md                 │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│          Row-Level Security Mapping               │
│  docs/rls_mapping.md                             │
│  sql/powerbi_target/rls_implementation.sql       │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│               Validation Controls                 │
│  scripts/run_validation.py                       │
│  sql/validation/                                 │
│  docs/validation_report.md                       │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│         Monday.com Operating Layer (modeled)      │
│  monday/board_schema.md                          │
│  monday/workflow_examples.md                     │
│  monday/automation_rules.md                      │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│              Stakeholder Visibility               │
│  docs/EXECUTIVE_SUMMARY.md                       │
│  docs/OPERATING_MODEL.md                         │
└──────────────────────────────────────────────────┘
```

---

## Pipeline Components

<p>
  <img src="https://img.shields.io/badge/Inventory-240%20Reports-0078D4?style=flat-square" alt="Inventory">
  <img src="https://img.shields.io/badge/Map-10%20Subject%20Areas-8b5cf6?style=flat-square" alt="Map">
  <img src="https://img.shields.io/badge/Transform-6%20Datasets-22c55e?style=flat-square" alt="Transform">
  <img src="https://img.shields.io/badge/Validate-1%25%20Threshold-f59e0b?style=flat-square" alt="Validate">
  <img src="https://img.shields.io/badge/Govern-Monday%20Model-F62B54?style=flat-square" alt="Govern">
  <img src="https://img.shields.io/badge/Serve-Power%20BI-F2C811?style=flat-square&logoColor=black" alt="Serve">
</p>

---

## Platform Components

| Component | Artifact | Purpose |
|---|---|---|
| Migration inventory | [config/migration_inventory.yaml](config/migration_inventory.yaml) | All 240 reports with status, owner, priority, and validation state |
| Semantic mapping | [docs/subject_area_mapping.md](docs/subject_area_mapping.md), [docs/rpd_to_dataflow_patterns.md](docs/rpd_to_dataflow_patterns.md) | OBIEE subject areas and RPD constructs mapped to Power BI datasets and DAX |
| Security and access mapping | [docs/rls_mapping.md](docs/rls_mapping.md), [sql/powerbi_target/rls_implementation.sql](sql/powerbi_target/rls_implementation.sql) | OBIEE catalog permissions translated to Power BI RLS roles with validation |
| Validation controls | [scripts/run_validation.py](scripts/run_validation.py), [sql/validation/](sql/validation/), [docs/validation_report.md](docs/validation_report.md) | Automated record count, aggregate, dimension, and security checks |
| Monday.com operating layer | [monday/](monday/) | Modeled workflow architecture for intake, change approval, issue triage, and stakeholder visibility |
| Data dictionary | [docs/data_dictionary.md](docs/data_dictionary.md) | Live field-level reference with business rule translations and accepted discrepancies |

---

## Monday.com Operational Orchestration Layer

Monday.com is modeled as the business-facing orchestration platform sitting above the technical migration framework. It governs the full platform lifecycle: intake, migration tracking, data-quality issue triage, change approvals, and steady-state operations.

**The model is a proposed workflow architecture, not a deployed production integration.**

| Board | Purpose |
|---|---|
| Report Intake Board | Structured intake and triage for new report requests and enhancements |
| Migration Status Board | Leadership-facing view of migration progress across all 240 reports |
| Data Quality Issue Board | Issue tracking from detection through resolution with owner accountability |
| Change Approval Board | Approval-gated change control for RLS, dataset, and measure changes |
| Platform Operations Board | Steady-state operations: access requests, refresh failures, incidents, enhancements |

See [monday/board_schema.md](monday/board_schema.md), [monday/workflow_examples.md](monday/workflow_examples.md), [monday/automation_rules.md](monday/automation_rules.md), and [monday/graphql_examples.md](monday/graphql_examples.md) for the full model.

---

## Migration Approach

### Phase 1: Inventory and mapping

Pulled the full inventory from the OBIEE RPD and catalog. Every subject area, every table, every column, every join, every security rule. Stored in [config/migration_inventory.yaml](config/migration_inventory.yaml) with status tracking per report.

Mapped the 10 OBIEE subject areas to 6 Power BI datasets. The consolidation eliminated 4 redundant subject areas that had been created as workarounds for RPD limitations. Mapping is in [docs/subject_area_mapping.md](docs/subject_area_mapping.md).

### Phase 2: Semantic layer conversion

The OBIEE RPD is a three-layer semantic model (physical, business model, presentation). Power BI does not have an equivalent. The conversion approach:

- Physical layer joins became Snowflake views (`sql/powerbi_target/dataset_views.sql`)
- Business model aggregation rules became DAX measures
- Presentation layer column aliases became Power BI field names
- RPD initialization blocks became Power Query parameters
- RPD session variables for security became Power BI RLS roles

Patterns are documented in [docs/rpd_to_dataflow_patterns.md](docs/rpd_to_dataflow_patterns.md).

### Phase 3: Security replication

OBIEE catalog security controlled which rows each user could see, based on group membership and data-level filters on the RPD. Translating this to Power BI RLS required:

1. Extracting every security rule from the OBIEE catalog
2. Mapping OBIEE groups to Power BI roles
3. Writing DAX filter expressions that replicate the OBIEE row filters
4. Testing each role against the OBIEE output to confirm identical row sets

The mapping and test results are in [docs/rls_mapping.md](docs/rls_mapping.md).

### Phase 4: Validation

Every one of the 240 reports was validated before go-live. The validation checked:

- Record counts (must match within 1%)
- Aggregate totals on all numeric measures (must match within 1%)
- Distinct dimension values (must be identical)
- Row-level security (same user must see same rows in both systems)

SQL for each check is in `sql/validation/`. The Python runner in `scripts/run_validation.py` automates the full suite and generates the report in [docs/validation_report.md](docs/validation_report.md).

### Phase 5: Data dictionary

Maintained a live data dictionary throughout migration as the single reference point. Every column mapping, every business rule translation, every known discrepancy. The dictionary is in [docs/data_dictionary.md](docs/data_dictionary.md).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Source BI | OBIEE 12c |
| Target BI | Power BI Desktop and Power BI Service |
| Transformation | Power Query M, DAX |
| Data warehouse | Snowflake |
| Validation | SQL, Python |
| Governance | Data dictionary, validation report, Monday.com modeled operating layer |
| Security | OBIEE catalog permissions mapped to Power BI Row-Level Security (RLS) |

---

## Validation

```bash
git clone https://github.com/nicholasjh-work/OBIEE-to-Power-BI-Conversion-Framework.git
cd OBIEE-to-Power-BI-Conversion-Framework
pip install -r requirements.txt
cp .env.example .env

# Run validation suite
python scripts/run_validation.py

# Run against a specific dataset
python scripts/run_validation.py --dataset finance

# Generate migration status report
python scripts/generate_mapping_report.py

# Run unit tests
pytest tests/
```

Validation checks cover row counts, aggregate variance, security mapping, and report-level reconciliation where artifacts are present.

---

## Project Structure

```
OBIEE-to-Power-BI-Conversion-Framework/
  config/                          Validation thresholds and migration inventory (240 reports)
  data/sample/                     Sample RPD metadata and report inventory extracts
  docs/                            Migration playbook, semantic mapping, RLS mapping, data dictionary
    EXECUTIVE_SUMMARY.md           Leadership-facing summary of platform and outcomes
    BUSINESS_SYSTEMS_OWNERSHIP.md  Enterprise business systems ownership capability map
    CAPABILITY_ALIGNMENT.md        Capability-to-function alignment reference
    OPERATING_MODEL.md             Operational framework for platform governance
    DATA_INTEGRITY_CONTROLS.md     Data quality controls and validation architecture
  monday/                          Modeled Monday.com operating layer
    board_schema.md                Board definitions, column types, and field mappings
    workflow_examples.md           End-to-end workflow examples per board
    automation_rules.md            Automation and notification rule patterns
    graphql_examples.md            Sample GraphQL queries and mutations
    sample_payloads/               Example JSON payloads for key workflow events
  powerquery/                      Reusable Power Query M patterns
  scripts/                         Python validation runner and mapping report generator
  sql/
    obiee_source/                  RPD metadata and catalog security extraction queries
    powerbi_target/                Snowflake views and RLS implementation
    validation/                    Record count, aggregate, dimension, and security audit checks
  tests/                           Pytest unit tests for validation logic
```

---

## Known Limitations

- The Monday.com content is a modeled operating layer, not a deployed production integration.
- Validation examples depend on available sample inputs and documented mappings. Results in `docs/validation_report.md` reflect the framework migration; sample data is illustrative.
- Power BI service deployment details are represented as framework artifacts, not live tenant configuration.
- The actual report definitions, RPD metadata, and proprietary business rules from the original engagement remain confidential. Sample data and configurations are illustrative.

---

<div align="center">
  <a href="https://www.linkedin.com/in/nicholasjhidalgo/">
    <img src="https://img.shields.io/badge/LinkedIn-Nicholas%20Hidalgo-0077B5?style=flat&logo=linkedin&logoColor=white" alt="LinkedIn">
  </a>
  &nbsp;
  <a href="https://nicholasjh-work.github.io">
    <img src="https://img.shields.io/badge/Portfolio-nicholasjh--work.github.io-0078D4?style=flat&logo=github&logoColor=white" alt="Portfolio">
  </a>
  &nbsp;
  <a href="mailto:nickhidalgo07@gmail.com">
    <img src="https://img.shields.io/badge/Email-nickhidalgo07%40gmail.com-EA4335?style=flat&logo=gmail&logoColor=white" alt="Email">
  </a>
</div>
