# Monday.com Operational Orchestration Layer

## What this is

This directory contains a modeled Monday.com workflow architecture for governing an enterprise BI platform. It is a portfolio implementation pattern, not a production deployment. The model demonstrates board schema design, automation logic, GraphQL integration patterns, and operator-facing workflow structure.

The technical BI platform migration is in the parent repository. Monday.com is modeled as the business-facing layer that sits on top of it: the surface through which stakeholders submit requests, the platform team tracks work, approvals are recorded, and leadership gets visibility.

## Why a workflow layer matters

An enterprise BI platform generates ongoing operational work that does not belong in GitHub or JIRA:

- Business users request new reports and enhancements
- Data quality issues surface from validation runs or user reports
- Changes to security roles, dataset views, or calculations need approval before deployment
- Leadership needs a status view that does not require reading YAML or SQL
- Access requests need an audit trail that is not email

Without a governed workflow layer, this coordination happens informally. Work gets lost, approvals are undocumented, and leadership visibility depends on someone remembering to send an update. Monday.com is the platform that structures these workflows into something auditable, trackable, and scalable.

## The five boards

| Board | Purpose |
|---|---|
| Report Intake Board | Structured intake for new report requests and enhancements |
| Migration Status Board | Leadership view of migration progress across all 240 reports |
| Data Quality Issue Board | Issue tracking from detection through resolution |
| Change Approval Board | Approval-gated change control for production platform changes |
| Platform Operations Board | Steady-state operations: access, refresh, incidents, enhancements |

## What is in this directory

| File | Contents |
|---|---|
| `board_schema.md` | Column definitions, item types, field mappings for all five boards |
| `workflow_examples.md` | End-to-end workflow examples showing how items move through each board |
| `automation_rules.md` | Automation and notification rule patterns for status changes, escalations, approvals |
| `graphql_examples.md` | Sample GraphQL queries and mutations for reading and writing board state |
| `sample_payloads/` | Example JSON payloads for key workflow events |

## How to read this model

Start with `board_schema.md` to understand the data model. Read `workflow_examples.md` to see how items move through the system. Read `automation_rules.md` to understand how the system responds to state changes without manual intervention. Read `graphql_examples.md` to understand how an external system (validation runner, monitoring tool, or integration layer) reads and writes board data programmatically.

The payloads in `sample_payloads/` represent the event surface: the moments when the system needs to communicate state changes to other systems or users.

## Relationship to the migration framework

The migration framework uses `config/migration_inventory.yaml` to track report status programmatically. The Migration Status Board in this model is the business-facing projection of that inventory: same data, readable by stakeholders who do not work in YAML files.

The validation runner in `scripts/run_validation.py` produces pass/fail results per report. In the operational model, failures from that runner create items on the Data Quality Issue Board automatically, via a webhook or scheduled GraphQL write.

The change approval model defines the gates that govern when a change produced in the technical framework can be deployed to production.
