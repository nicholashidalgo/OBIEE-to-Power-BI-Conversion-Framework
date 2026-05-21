# Platform Operating Model

## Purpose

This document describes the operating model for an enterprise BI platform after the initial migration is complete. It covers the governance structure, workflow layer, operational responsibilities, and escalation paths that keep the platform running reliably and evolving in a controlled way.

The migration framework in this repo handles the technical conversion. This document handles what comes after: steady-state governance.

## The two phases of platform ownership

**Phase 1: Migration**
Inventory, convert, validate, secure, and cut over. The migration playbook in `docs/migration_playbook.md` governs this phase.

**Phase 2: Operations**
Intake new requests, triage issues, approve changes, manage access, and maintain data quality. This document governs Phase 2.

## Workflow layer

Monday.com serves as the business-facing orchestration layer. It is the surface through which stakeholders interact with the platform team. All requests, issues, changes, and approvals move through Monday.com boards. The platform team operates from these boards. Leadership visibility comes from board-level rollups and dashboards.

The five boards and their functions:

### Report Intake Board

Governs how new report requests enter the platform backlog.

- Business users submit requests through a structured intake form
- Each submission captures: requestor, business owner, use case, source dataset, priority justification, expected user count, and deadline if applicable
- The platform team reviews each submission before it enters the build queue
- Intake review confirms dataset alignment, security requirements, and delivery estimate
- Approved requests move to the Migration Status Board or the enhancement backlog
- Rejected or deferred requests receive a written disposition and can be resubmitted

**Key governance rule:** No report enters the build queue without a completed intake review. Ad hoc requests received outside the intake board are redirected to the form.

### Migration Status Board

Provides leadership and stakeholders with a live view of platform migration progress.

- One item per report, mirroring `config/migration_inventory.yaml`
- Grouped by dataset (Finance, Risk, Operations)
- Status phases: Pending, In Progress, Validation, Stakeholder Review, Migrated, Blocked
- Each item links to the relevant validation result and data dictionary entry
- Weekly status rollup sent automatically to platform leadership
- Blocked items surface in a dedicated filter view with owner and blocker documented

**Key governance rule:** Status is updated by the platform team, not by requestors. Status changes require validation evidence or a documented exception.

### Data Quality Issue Board

Tracks every data quality finding from detection through resolution.

- Issues are created automatically from validation failures and manually from business-reported discrepancies
- Each issue captures: affected dataset, affected reports, detection source, severity, root-cause category, owner, and resolution status
- Severity levels: Critical (blocking), High (resolution required before next release), Medium (tracked), Low (informational)
- Issues do not close until the relevant validation check passes again
- High and Critical issues trigger automatic notifications to the dataset owner
- Issues open for more than 14 days without a resolution plan escalate to platform leadership

**Root-cause categories:** Source data, migration logic, RLS filter, measure calculation, dimension mapping, refresh timing, other.

**Key governance rule:** No data quality issue is closed by assertion. Every closure requires a passing validation run or a written acceptance of known discrepancy with business owner sign-off.

### Change Approval Board

Governs all changes to the production platform that affect data definitions, security, or report behavior.

- Changes to RLS roles, dataset views, measure definitions, or Power Query transforms require a change request before implementation
- Each request captures: change description, affected datasets, affected reports, downstream impact assessment, and requested approver
- Approval tiers:
  - Tier 1 (cosmetic or additive): platform team lead approval
  - Tier 2 (logic or calculation change): dataset owner and business stakeholder approval
  - Tier 3 (RLS or security change): platform lead, security reviewer, and affected business owner approval
- A deployment gate holds the change in the board until all required approvals are recorded
- Emergency changes follow an expedited path with post-deployment documentation required within 24 hours

**Key governance rule:** No change is deployed to production without approval recorded in the board. Approval in Slack or email is not sufficient.

### Platform Operations Board

Steady-state view of platform health and pending operational work.

- Tracks: access requests, refresh failures, Power BI service incidents, RLS audit findings, and enhancement requests
- Access requests capture: requestor, role requested, business justification, and approver
- Refresh failures auto-create items via monitoring integration
- Enhancement requests are triaged weekly and moved to the intake board if they qualify as new reports
- Platform health metrics are surfaced in a board dashboard: refresh success rate, open access requests, open issues by severity, change approvals in flight

## Escalation paths

| Situation | Escalation |
|---|---|
| Validation failure on a report in production | Data Quality Issue Board, severity Critical, notify dataset owner immediately |
| Security change with no approval record | Block deployment, create Change Approval item, notify platform lead |
| Report request submitted outside intake | Redirect to intake form, do not begin work |
| Data Quality Issue open 14+ days without resolution plan | Auto-escalate to platform leadership with issue summary |
| Refresh failure affecting a Critical report | Incident item on Operations Board, notify business owner within 1 hour |

## Access management

All access requests flow through the Platform Operations Board. Access is never granted by verbal or email request alone.

- Each request documents: user, role requested, business justification, manager approval, and effective date
- RLS role assignments are reviewed quarterly against the current user list
- Terminated users are removed from all Power BI roles within 24 hours of offboarding notification
- Quarterly access audit results are documented in the Operations Board with any remediation items tracked to closure

## Data dictionary governance

The data dictionary in `docs/data_dictionary.md` is the authoritative reference for all field definitions, business rule translations, and known discrepancies.

- Any change to a dataset view or measure definition requires a data dictionary update before deployment
- Known discrepancies are reviewed quarterly with business owners
- New fields added during enhancements are documented before the enhancement is marked complete
