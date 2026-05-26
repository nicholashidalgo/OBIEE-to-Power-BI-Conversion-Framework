# Release Management and User Acceptance Testing Model

## 1. Purpose

This document defines the release governance model and user acceptance testing (UAT) process for a BI platform migration. It covers release planning, test design, stakeholder roles, defect classification, sign-off, release notes, deployment, rollback, hypercare, and production readiness criteria.

The model is role-based and company-neutral. It is designed to govern any cohort-based BI platform release where validated reports must be accepted by business users before go-live. It is not a software deployment framework—it is a data platform release framework, where the primary quality concern is data accuracy, security integrity, and user workflow continuity.

---

## 2. Business Use Case

A validation suite confirms that Power BI report data matches the source system. UAT confirms that business users—working with their own data, in their own workflows—find the reports usable and accurate for real decisions. These are different tests. Validation catches technical discrepancies. UAT catches business logic gaps, usability problems, and cases where the report structure is technically correct but doesn't support the actual use case.

Release management governs the sequencing of reports from validated to live: which cohort goes first, what the rollback window is, who signs off, and what support model is in place during the transition period. Without this structure, releases create chaos: users discover issues post-cutover, the platform team has no documented basis for accepting or rejecting a defect report, and rollback decisions become political rather than technical.

---

## 3. Users and Stakeholders

| Role | Release Responsibility |
|---|---|
| Platform Lead | Authorizes release cohorts; owns production readiness checklist; authorizes rollback |
| Dataset Owner | Signs off on dataset-level UAT completion; escalation point for data defects |
| Business Owner (per report or cohort) | Participates in UAT; signs off that the report meets business requirements |
| Platform Engineer | Resolves defects found during UAT; re-validates after fixes |
| Hypercare Support | Provides elevated support coverage during the hypercare period |
| Change Management Lead | Coordinates user communications, training, and adoption activities |
| End User (UAT Participant) | Executes test scenarios; raises defects; confirms resolution |

---

## 4. Migration Scope

This release model governs the deployment of:

- **240 reports** released in phased cohorts across Finance, Risk, and Operations datasets
- Each cohort aligned to a Power BI dataset (Finance first, Risk second, Operations third—sequenced by business priority and dependency)
- Reports within a cohort released together with a shared cutover date
- **30-day parallel run** period per cohort: OBIEE retained in read-only mode while users validate the new platform

The 30-day parallel run is a critical risk control. Both systems read from the same Snowflake warehouse. Users who find an issue in Power BI can check OBIEE to confirm whether the difference is a migration problem or a source data timing issue. This comparison capability reduces support volume and user anxiety during the transition.

---

## 5. Operating Model

### Release Phases

**Phase 1: Pre-release readiness**
- Validation suite passed for all reports in the cohort (see [`docs/migration-governance/REPORT_RECONCILIATION_AND_PARITY_TESTING.md`](REPORT_RECONCILIATION_AND_PARITY_TESTING.md))
- Dataset certification complete (see [`docs/migration-governance/SEMANTIC_DATASET_CERTIFICATION.md`](SEMANTIC_DATASET_CERTIFICATION.md))
- UAT test scenarios documented and test participants identified
- Release notes drafted
- Rollback plan confirmed

**Phase 2: UAT**
- UAT participants execute test scenarios using the migrated Power BI environment
- Defects raised and classified
- Critical and High defects resolved before sign-off
- Business owner sign-off collected

**Phase 3: Release authorization**
- Production readiness checklist reviewed by Platform Lead
- Release notes finalized
- Go-live date confirmed with business owners
- Communication sent to all affected users

**Phase 4: Cutover**
- Users directed to Power BI
- OBIEE retained in read-only mode (30-day window)
- Hypercare period begins

**Phase 5: Hypercare and stabilization**
- Elevated support coverage for first 14 days post-cutover
- Issues triaged within same-day SLA during hypercare
- Post-cutover validation re-run after first scheduled refresh
- Hypercare formally closed at 14-day mark with documented open items

**Phase 6: OBIEE decommission**
- At day 30, if no Critical or unresolved High issues exist, OBIEE is decommissioned for the completed cohort
- Decommission is documented in the Migration Status Board
- Any remaining parallel access is removed from user training materials and communications

### Cohort Release Model

Releasing all 240 reports simultaneously creates unmanageable support volume and limits rollback options. A cohort-based model sequences releases by dataset, grouping reports that share a common semantic layer and user base.

| Cohort | Dataset | User Groups | Release Sequence |
|---|---|---|---|
| Cohort 1 | Finance | FP&A, AP Team, AR Team | First |
| Cohort 2 | Risk | Risk Team | Second |
| Cohort 3 | Operations | Supply Chain, Logistics, Quality | Third |

Within each cohort, high-priority reports (as defined in `config/migration_inventory.yaml`) are released first in the UAT cycle to maximize stakeholder coverage on the most-used reports.

### Workflow Integration

The Monday.com workflow model (documented in `monday/`) supports the release process through:

- **Migration Status Board:** tracks each report through Validation → Stakeholder Review → Migrated, providing the release team with a live view of cohort readiness
- **Change Approval Board:** gates any fix deployed during UAT or hypercare through the same change approval process as production changes
- **Data Quality Issue Board:** captures all defects raised during UAT and hypercare with owner assignment and resolution tracking
- **Platform Operations Board:** tracks the hypercare support workload and access requests from newly onboarded users

**Operating model status:** The Monday.com board layer is a designed workflow model, not a deployed production integration. The board schemas and workflow examples in `monday/` are implementable designs.

---

## 6. Architecture

### UAT Environment Design

UAT should run against a representation of the production data environment. Two approaches are common:

**Approach A: Shared production-equivalent environment**
UAT participants access a separate Power BI workspace that reads from the same Snowflake views as production. Data is production data. No synthetic data is used in UAT.

**Approach B: Production environment with limited access**
UAT participants are granted preview access to the production workspace for the duration of UAT. Go-live removes the preview designation.

Both approaches ensure UAT validates against real data, not synthetic values. The risk of Approach A is that issues found in UAT may reveal actual data problems that users then see. This is acceptable—finding them before go-live is the point.

### Defect Classification Architecture

All defects raised during UAT follow the same four-category classification used in technical validation:

| Classification | Definition | UAT Resolution Path |
|---|---|---|
| Migration error | Report produces incorrect results due to conversion logic | Platform engineer fixes; re-validate; re-test in UAT |
| Source data issue | Report shows a difference from OBIEE due to a pre-existing OBIEE defect | Document; confirm Power BI behavior is correct; business owner accepts |
| Business logic gap | Report is technically correct but does not meet actual business need | Triage: is this a scope change or a bug? Scope changes go to intake; bugs go to engineering |
| Usability issue | Data is correct but report layout, filter design, or navigation is problematic | Platform engineer addresses; design change goes through Tier 1 approval |

A **Critical defect** (security gap or data accuracy failure that produces wrong decisions) blocks release until resolved.
A **High defect** (data accuracy failure with a known workaround) requires a documented resolution plan before sign-off, but may allow conditional sign-off if the business owner accepts the risk and a resolution date is committed.
A **Medium or Low defect** is tracked and targeted for resolution in a future release cycle. Sign-off is not blocked.

---

## 7. Controls

### Production Readiness Checklist

Before any cohort is authorized to release, the following must be confirmed:

**Technical readiness**
- [ ] Validation suite passed for all reports in cohort (pass rate and any accepted discrepancies documented)
- [ ] Dataset certified per `docs/migration-governance/SEMANTIC_DATASET_CERTIFICATION.md`
- [ ] All Critical and High technical defects resolved and re-validated
- [ ] Post-fix validation re-run and results documented
- [ ] RLS roles validated for all user groups in the cohort
- [ ] Scheduled refresh confirmed and tested

**UAT readiness**
- [ ] All UAT test scenarios executed
- [ ] All Critical UAT defects resolved
- [ ] High UAT defects: resolved or accepted with documented resolution plan
- [ ] Business owner sign-off collected for each report or report group in cohort
- [ ] Sign-off documented in Migration Status Board

**Release readiness**
- [ ] Release notes complete and reviewed
- [ ] User communication drafted and approved
- [ ] Training materials available (quick-start guide, report naming map, support contact)
- [ ] Rollback plan confirmed: OBIEE read-only access still available, rollback trigger criteria defined
- [ ] Hypercare support schedule confirmed (who, how long, how to reach)
- [ ] Platform Operations Board items created for hypercare monitoring

### Change Control During UAT and Hypercare

Any fix deployed during UAT or hypercare is treated as a production change. It goes through the Change Approval Board. There are no informal hotfixes.

- Tier 1 approval required for cosmetic fixes (label changes, formatting)
- Tier 2 approval required for calculation or measure changes
- Tier 3 approval required for any RLS modification, even a minor one

Emergency changes (a Critical issue that requires immediate action) follow the expedited path: deploy first, document within 24 hours. Platform Lead must authorize emergency deploys.

---

## 8. Validation Workflow

### UAT Test Scenario Design

UAT test scenarios are designed per report and capture the business user's actual workflow, not just a data comparison.

A UAT test scenario includes:

- **Report name and cohort**
- **Test participant role** (business title and data access level)
- **Scenario description** (what business question the user is answering)
- **Steps** (navigate to report, apply filters, drill to detail)
- **Expected result** (what number or list the user expects to see)
- **Comparison method** (side-by-side with OBIEE during parallel run, or confirmation from business knowledge)
- **Pass/fail criteria**

Example UAT scenario structure:

| Field | Example |
|---|---|
| Report | AR Aging Report |
| Participant | AR Analyst (Finance_NA role) |
| Scenario | Pull current aging bucket totals for North America as of today |
| Steps | Open report; confirm North America filter is applied via RLS; review 30-day, 60-day, 90-day, 90+ buckets |
| Expected result | Bucket totals consistent with OBIEE output for the same date range; Finance_EMEA data not visible |
| Pass criteria | Bucket totals within 1%; EMEA rows absent |

### UAT Execution and Defect Logging

UAT participants record results in the shared tracking system (Migration Status Board for pass results; Data Quality Issue Board for defects). Each defect entry captures:

- Report name
- Description of the discrepancy
- Screenshot or data export showing the issue
- OBIEE comparison result (if available)
- Severity assessment (participant's initial view; platform team classifies formally)
- Date raised

Platform team responds to defects within:
- Critical: same business day
- High: within 2 business days
- Medium/Low: tracked for future release

### Sign-off Process

Business owner sign-off is collected per cohort. A single sign-off can cover multiple related reports if the business owner has reviewed a representative set. Sign-off documents:

- Business owner name and role
- Reports or report group covered
- Date of sign-off
- Any conditional acceptances (High defects accepted with documented resolution dates)
- Sign-off recorded in Migration Status Board (Stakeholder Signoff column set to Signed Off)

---

## 9. Release Workflow

```
Validation suite complete for cohort
        │
        ▼
Dataset certification confirmed
        │
        ▼
UAT period begins (defined window: recommend 5–10 business days per cohort)
        │
        ├── Defects raised → classified → resolved or accepted
        │
        ▼
All Critical defects resolved
High defects resolved or accepted with plan
        │
        ▼
Business owner sign-off collected
        │
        ▼
Production readiness checklist reviewed (Platform Lead)
        │
        ▼
Release notes finalized
User communication sent (minimum 5 business days before cutover)
        │
        ▼
Cutover date
Users directed to Power BI
OBIEE retained read-only (30-day window)
Hypercare begins
        │
        ▼
Day 7: Hypercare mid-point review
Open issues logged in Platform Operations Board
        │
        ▼
Day 14: Formal hypercare close
Remaining open items tracked in normal support queue
        │
        ▼
Day 30: OBIEE decommission (if no Critical or unresolved High issues)
Post-decommission confirmation logged
```

### Rollback Criteria

Rollback is initiated if any of the following occur during the hypercare period:

- A Critical data accuracy failure is confirmed that affects a high-volume report and cannot be resolved within 24 hours
- A security failure is confirmed: a user can access rows they should not be able to see
- Scheduled refresh fails for more than 48 consecutive hours with no confirmed resolution

Rollback action: users are directed back to OBIEE for the affected reports. Power BI remains available for unaffected reports. The issue is investigated and resolved in Power BI before a re-cutover is attempted.

Rollback does not trigger a re-release cycle from scratch. Only the affected reports revert. The 30-day decommission clock for unaffected reports continues.

---

## 10. Operational Metrics

| Metric | Description |
|---|---|
| UAT defect volume by severity | Count of Critical/High/Medium/Low defects raised during UAT |
| Defect resolution rate before sign-off | % of Critical and High defects resolved before business owner sign-off |
| Time from UAT open to sign-off | Elapsed days per cohort |
| Hypercare defect volume | Count of new issues raised post-cutover during hypercare period |
| Rollback events | Count of rollbacks during hypercare; target is 0 |
| Days to OBIEE decommission | Elapsed days from cutover to decommission; target is ≤30 |
| User-reported issues in first 30 days | Count of issues raised by end users post-go-live |

---

## 11. Workplace Application

This release model is applicable to any phased BI platform migration where:

- Reports must be validated before being shown to users
- Business users participate in acceptance testing before go-live
- A parallel run period allows comparison during the transition
- The platform team needs a documented rollback capability

**The model adapts to different organizational contexts:**

- In organizations with a formal change advisory board (CAB), the production readiness checklist maps to CAB submission requirements
- In organizations with an existing UAT framework (QA team, ITSM-managed UAT process), the defect classification and sign-off steps integrate into the existing workflow rather than replacing it
- The cohort sequencing model applies equally to Cognos, Tableau, MicroStrategy, or Business Objects migrations—the underlying principle is the same: sequence by risk, validate before release, maintain a fallback

---

## 12. Limitations

- The UAT model described here assumes the business users participating in UAT have access to OBIEE for side-by-side comparison during the parallel run period. If the source system is decommissioned before UAT, the comparison basis is the validation report outputs rather than live system access.
- The release timing guidance (5–10 business days for UAT per cohort) is a reference range, not a requirement. Actual timing depends on report count, user availability, and defect volume.
- The Monday.com workflow integration is modeled, not deployed. The defect tracking and sign-off steps described here assume a workflow board is in place. In the absence of Monday.com, equivalent steps can be executed in Jira, ServiceNow, or a shared tracking spreadsheet.
- This model does not cover Power BI Service workspace configuration, deployment pipeline setup (Dev → Test → Prod), or sensitivity label management. Those are platform-level deployment controls beyond the scope of this documentation framework.

---

## 13. What This Does Not Claim

- This document does not claim that a live release process executed against this repo's artifacts is in progress or has been executed. The model is a designed release governance framework.
- The hypercare support model described does not imply a specific staffing level or SLA commitment. Reference targets are provided as guidance, not contractual obligations.
- The rollback capability described depends on OBIEE remaining available in read-only mode during the parallel run. If the source system was decommissioned at migration start, the rollback capability is limited to restoring from the last known good Power BI state rather than reverting users to OBIEE.

---

## 14. Extension Path

- **Automated release readiness reporting:** Extend `scripts/generate_mapping_report.py` to output a release readiness report showing which reports are in Validation, Stakeholder Review, or Migrated status, with UAT sign-off percentage per cohort
- **UAT defect tracker integration:** Connect the defect classification workflow to the Data Quality Issue Board via Monday.com GraphQL mutations (patterns in `monday/graphql_examples.md`), automatically creating issue board items from UAT defect forms
- **Release notes automation:** Generate structured release notes from `config/migration_inventory.yaml` by filtering for reports changing status to `migrated` in a given release window, with OBIEE-to-Power-BI name mapping
- **Deployment pipeline:** Add a Power BI deployment pipeline (Dev → Test → Production workspace) so that UAT occurs in the Test workspace and promotion to Production is the release action, replacing manual workspace management

---

## 15. Interview Talking Points

**On why UAT is separate from validation:**
> "Technical validation confirms the numbers are right. UAT confirms the report is usable. Those are different tests. Validation can't tell you that a business analyst's actual workflow requires a filter that isn't available, or that the report they're replacing had a prompt that they depended on and the new report doesn't have it. Those gaps surface in UAT with real users doing real work."

**On the cohort release model:**
> "Releasing all 240 reports at once creates an unmanageable support situation. With a cohort model, you release Finance first, learn from that experience, and apply it to Risk and Operations. The support volume is predictable, the rollback scope is contained, and you're not asking the entire organization to change at once."

**On the rollback plan:**
> "The rollback plan is the thing that lets you be decisive during go-live. If a Critical issue surfaces post-cutover, you don't deliberate—you execute the rollback plan. Both systems read from the same Snowflake warehouse, so no data is lost and no ETL reversal is needed. Users go back to OBIEE for those reports while the fix is made. The 30-day parallel run window exists precisely for this scenario."

**On defect classification during UAT:**
> "The business logic gap category is the one that causes the most friction. A user says 'this is wrong,' but what they mean is 'this doesn't match my expectation.' Sometimes the report is correct and the expectation was formed by an OBIEE behavior that was actually a bug. Sometimes the report is technically correct but doesn't serve the actual use case. Figuring out which one it is requires the defect classification conversation, not just a ticket."

**On change control during UAT:**
> "Every fix deployed during UAT goes through the change approval process. There are no informal hotfixes. The reason is audit integrity: if you find a regression six months later, you need to know exactly what changed during UAT and when. An undocumented fix that resolved a UAT defect is invisible in your change history."
