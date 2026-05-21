# Monday.com Board Schema

## Board 1: Report Intake Board

**Purpose:** Structured intake and triage for new report requests and enhancement requests.

**Groups:**
- New Submissions (unreviewed)
- In Triage (under platform team review)
- Approved - In Backlog
- Deferred
- Rejected

**Columns:**

| Column | Type | Description |
|---|---|---|
| Item Name | Text | Report or enhancement name |
| Request Type | Status | New Report, Enhancement, Access Change, Other |
| Requestor | Person | Submitting user |
| Business Owner | Person | Accountable stakeholder for the request |
| Use Case | Long Text | Business justification and intended audience |
| Source Dataset | Dropdown | Finance, Risk, Operations, New Dataset Required |
| Priority | Status | Critical, High, Medium, Low |
| Priority Justification | Long Text | Why this priority level |
| Expected User Count | Number | Estimated report consumers |
| Deadline | Date | Business-stated deadline if applicable |
| Intake Review Status | Status | Pending, In Review, Approved, Deferred, Rejected |
| Reviewer | Person | Platform team member conducting triage |
| Review Notes | Long Text | Disposition notes, questions, or blockers |
| Dataset Alignment Confirmed | Checkbox | Reviewer confirms dataset target is appropriate |
| Security Requirements | Long Text | RLS roles required or changes needed |
| Delivery Estimate | Date | Platform team estimate after triage |
| Linked Migration Item | Board Relation | Link to Migration Status Board item if applicable |
| Submission Date | Date | Auto-populated on item creation |
| Last Updated | Last Updated | Auto-populated |

**Intake governance rules:**
- Items remain in New Submissions until a reviewer is assigned
- Intake review must be completed within 3 business days of submission
- Rejected items receive a written reason in Review Notes before status is set
- Deferred items have a revisit date set in the Delivery Estimate field

---

## Board 2: Migration Status Board

**Purpose:** Leadership-facing view of migration progress across all 240 reports.

**Groups:**
- Finance Dataset
- Risk Dataset
- Operations Dataset
- Blocked
- Decommissioned (post-cutover cleanup)

**Columns:**

| Column | Type | Description |
|---|---|---|
| Item Name | Text | Report name |
| Legacy System | Text | OBIEE subject area |
| Target Dataset | Dropdown | Finance, Risk, Operations |
| Migration Status | Status | Pending, In Progress, Validation, Stakeholder Review, Migrated, Blocked |
| Owner | Person | Platform team member responsible |
| Priority | Status | High, Medium, Low |
| Validation Status | Status | Not Started, Running, Passed, Failed, Accepted Discrepancy |
| Validation Run Date | Date | Date of last validation run |
| Stakeholder Signoff | Status | Not Started, Pending, Signed Off |
| Signoff Owner | Person | Business owner providing sign-off |
| Blocker Description | Long Text | Required when status is Blocked |
| Go Live Date | Date | Actual or target cutover date |
| OBIEE Decommission Date | Date | Date OBIEE version is retired |
| Linked Intake Item | Board Relation | Link to originating intake request |
| Linked DQ Issues | Board Relation | Link to open Data Quality Issue items |
| Notes | Long Text | Migration-specific notes or known discrepancies |
| Last Updated | Last Updated | Auto-populated |

**Status transition rules:**
- Pending to In Progress: owner assigned, work started
- In Progress to Validation: conversion complete, validation suite run initiated
- Validation to Stakeholder Review: all validation checks passed
- Stakeholder Review to Migrated: business owner sign-off recorded
- Any status to Blocked: blocker description required

---

## Board 3: Data Quality Issue Board

**Purpose:** Track every data quality finding from detection through resolution with owner accountability.

**Groups:**
- Critical (open)
- High (open)
- Medium (open)
- Low / Informational
- Resolved
- Accepted Discrepancy (known, documented, signed off)

**Columns:**

| Column | Type | Description |
|---|---|---|
| Item Name | Text | Brief issue description |
| Detection Source | Dropdown | Validation Run, User Report, Access Audit, Scheduled Check, Other |
| Affected Dataset | Dropdown | Finance, Risk, Operations, Cross-Dataset |
| Affected Reports | Long Text | Report names or IDs impacted |
| Severity | Status | Critical, High, Medium, Low |
| Root Cause Category | Dropdown | Source Data, Migration Logic, RLS Filter, Measure Calculation, Dimension Mapping, Refresh Timing, Other |
| Owner | Person | Accountable for resolution |
| Status | Status | Open, In Investigation, Resolution in Progress, Pending Validation, Resolved, Accepted |
| Detection Date | Date | When the issue was first identified |
| Target Resolution Date | Date | Committed resolution date |
| Resolution Date | Date | Actual resolution date |
| Resolution Description | Long Text | What changed to resolve the issue |
| Validation Evidence | Long Text | Reference to passing validation run or acceptance documentation |
| Business Owner Signoff | Status | Not Required, Pending, Signed Off |
| Signoff Owner | Person | Business owner for acceptance sign-off |
| Linked Migration Item | Board Relation | Link to affected Migration Status Board items |
| Linked Change Request | Board Relation | Link to Change Approval Board if a code change is required |
| Last Updated | Last Updated | Auto-populated |

**Resolution governance rules:**
- Critical issues trigger immediate notification to dataset owner and platform lead
- Issues do not move to Resolved without validation evidence or acceptance documentation
- Issues open more than 14 days without a resolution plan auto-escalate
- Accepted Discrepancy status requires business owner sign-off and data dictionary update

---

## Board 4: Change Approval Board

**Purpose:** Approval-gated change control for all production changes that affect data definitions, security, or report behavior.

**Groups:**
- Pending Submission (draft)
- Awaiting Tier 1 Approval
- Awaiting Tier 2 Approval
- Awaiting Tier 3 Approval
- Approved - Pending Deployment
- Deployed
- Rejected
- Emergency Changes

**Columns:**

| Column | Type | Description |
|---|---|---|
| Item Name | Text | Change title |
| Change Type | Dropdown | RLS Role Change, Dataset View Change, Measure Definition, Power Query Transform, Schema Change, Access Grant, Other |
| Approval Tier | Status | Tier 1, Tier 2, Tier 3, Emergency |
| Change Description | Long Text | What is changing and why |
| Affected Datasets | Long Text | Datasets impacted |
| Affected Reports | Long Text | Reports impacted |
| Downstream Impact Assessment | Long Text | What business behavior changes and for whom |
| Requestor | Person | Who is requesting the change |
| Tier 1 Approver | Person | Platform team lead |
| Tier 1 Status | Status | Pending, Approved, Rejected |
| Tier 2 Approver | Person | Dataset owner and business stakeholder |
| Tier 2 Status | Status | Not Required, Pending, Approved, Rejected |
| Tier 3 Approver | Person | Security reviewer or additional required approver |
| Tier 3 Status | Status | Not Required, Pending, Approved, Rejected |
| Deployment Gate | Status | Held, Released |
| Requested Deploy Date | Date | Requested deployment window |
| Actual Deploy Date | Date | Actual deployment date |
| Post-Deploy Validation | Status | Not Started, Running, Passed, Failed |
| Linked DQ Issue | Board Relation | Link to originating Data Quality Issue if applicable |
| Rollback Plan | Long Text | Steps to revert if deployment fails |
| Last Updated | Last Updated | Auto-populated |

**Approval tier definitions:**
- Tier 1: Cosmetic or additive changes (new column, label rename, non-breaking addition)
- Tier 2: Logic or calculation changes affecting measure values or aggregation behavior
- Tier 3: RLS role changes, security filter modifications, access grants affecting row visibility

**Deployment gate rule:** Deployment Gate remains Held until all required approval statuses are set to Approved. Gate release is automated based on approval status values.

---

## Board 5: Platform Operations Board

**Purpose:** Steady-state operations view for access requests, refresh monitoring, incidents, and enhancements.

**Groups:**
- Access Requests (open)
- Refresh Failures (active)
- Incidents (open)
- Enhancement Queue
- Closed / Resolved

**Columns:**

| Column | Type | Description |
|---|---|---|
| Item Name | Text | Operation item title |
| Item Type | Dropdown | Access Request, Refresh Failure, Incident, Enhancement Request, RLS Audit Finding, Other |
| Priority | Status | Critical, High, Medium, Low |
| Status | Status | Open, In Progress, Pending Approval, Resolved, Closed |
| Owner | Person | Accountable platform team member |
| Requestor | Person | Who raised the item |
| Dataset Affected | Dropdown | Finance, Risk, Operations, Platform-Wide |
| Description | Long Text | Full description of the request or issue |
| Business Justification | Long Text | For access requests and enhancements |
| Manager Approval | Status | Not Required, Pending, Approved |
| Manager | Person | For access requests |
| Resolution | Long Text | What was done to resolve the item |
| Open Date | Date | When the item was created |
| Close Date | Date | When the item was resolved |
| Linked Change Request | Board Relation | Link to Change Approval Board if a change is required |
| Last Updated | Last Updated | Auto-populated |
