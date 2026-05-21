# Monday.com Workflow Examples

## Workflow 1: New Report Request

**Scenario:** A business analyst in the Finance team needs a new cash flow summary report that does not exist in the current Power BI environment.

**Step 1: Intake submission**
The analyst opens the Report Intake Board and creates a new item in the New Submissions group. They complete all required fields: use case, source dataset (Finance), priority (High), expected user count (12), and business owner (Finance Director). Submission date is auto-populated.

**Step 2: Triage notification**
An automation triggers on item creation in New Submissions and sends a notification to the platform team channel: "New intake submission: Cash Flow Summary Report -- Finance Dataset -- Priority: High. Review required within 3 business days."

**Step 3: Intake review**
A platform team member is assigned as Reviewer. They set status to In Review. They confirm dataset alignment (Finance dataset can support the request), document the security requirements (Finance_NA and Finance_Global roles), and set a delivery estimate.

**Step 4: Approval and backlog entry**
Reviewer sets Intake Review Status to Approved. An automation moves the item to the Approved - In Backlog group and creates a linked item on the Migration Status Board with status Pending.

**Step 5: Build and validation**
When the platform team begins work, they update the Migration Status Board item to In Progress. After the view and report are built, they run the validation suite and update Validation Status to Passed.

**Step 6: Stakeholder review and sign-off**
Status moves to Stakeholder Review. The business owner receives an automated notification: "Cash Flow Summary Report is ready for your review and sign-off. Please confirm the numbers match your expectations." When the business owner confirms, they update Stakeholder Signoff to Signed Off. Status moves to Migrated.

---

## Workflow 2: Data Quality Issue

**Scenario:** The overnight validation run detects a record count variance of 2.3% on the Finance AR Aging Detail report, which exceeds the 1% threshold.

**Step 1: Issue creation**
The validation runner script writes a new item to the Data Quality Issue Board via GraphQL mutation. Fields populated: Item Name (AR Aging Detail -- Record Count Variance 2.3%), Detection Source (Validation Run), Affected Dataset (Finance), Severity (High), Status (Open), Detection Date (today).

**Step 2: Owner notification**
An automation triggers on item creation with Severity = High and sends an immediate notification to the Finance dataset owner: "High-severity data quality issue created: AR Aging Detail record count variance 2.3% (threshold: 1%). Investigation required."

**Step 3: Investigation**
Owner updates status to In Investigation. After reviewing the SQL validation output and the source data, they identify the root cause: a batch of reversed entries posted in the source system this morning that OBIEE included in its count but the Snowflake view correctly excludes. Root Cause Category is set to Source Data.

**Step 4: Resolution path**
Because the discrepancy is a source data issue and the Power BI behavior is correct, the resolution is documentation rather than a code change. Owner updates Resolution Description, sets Business Owner Signoff to Pending, and notifies the Finance Director.

**Step 5: Acceptance and closure**
The Finance Director confirms the source system issue is known and accepts the Power BI behavior as correct. Business Owner Signoff is set to Signed Off. The data dictionary is updated to document this discrepancy. Status is set to Accepted. The platform team also updates `docs/data_dictionary.md` with the accepted discrepancy before closing the item.

---

## Workflow 3: RLS Role Change

**Scenario:** The Risk team is reorganizing. One of their regional groups is being dissolved, and affected users need to be reassigned to a different Power BI RLS role.

**Step 1: Change request**
A platform team member creates a new item on the Change Approval Board. Change Type: RLS Role Change. Approval Tier: Tier 3. They complete the change description, list the affected users (14), document the downstream impact (users will see a different row set in the Risk Operational report), and tag the security reviewer as Tier 3 Approver.

**Step 2: Deployment gate held**
Deployment Gate is automatically set to Held on item creation. No deployment can proceed until all required approvals are recorded.

**Step 3: Tier 1 approval**
The platform team lead reviews the change. Confirms the DAX filter change is technically correct. Sets Tier 1 Status to Approved.

**Step 4: Tier 2 approval**
The Risk dataset owner and business stakeholder review the impact assessment. The business stakeholder confirms the reassignment is correct and intentional. Sets Tier 2 Status to Approved.

**Step 5: Tier 3 approval**
The security reviewer confirms the new role assignment does not grant users access to data they are not entitled to see. Reviews the access audit for the affected users. Sets Tier 3 Status to Approved.

**Step 6: Gate release and deployment**
With all three approval statuses set to Approved, an automation sets Deployment Gate to Released and notifies the platform team: "Change approved. Deployment permitted. Requested deploy date: [date]."

**Step 7: Post-deployment validation**
After deployment, the platform team runs the security audit SQL for the affected roles and updates Post-Deploy Validation to Passed. Actual Deploy Date is recorded. The RLS mapping documentation in `docs/rls_mapping.md` is updated.

---

## Workflow 4: Escalation on Stale Issue

**Scenario:** A High-severity data quality issue has been open for 15 days with no resolution plan documented.

**Step 1: Escalation trigger**
An automation runs nightly and checks for items where Severity = High or Critical and Status is not Resolved or Accepted and Open Date is more than 14 days ago. The AR Aging Detail source data issue meets this condition (it was accepted but a similar new issue was opened and not acted on).

**Step 2: Escalation notification**
The automation sends a notification to the platform lead: "Escalation: 1 High-severity data quality issue has been open for 15+ days without a resolution plan. Issue: [item name]. Owner: [name]. Please review."

**Step 3: Platform lead review**
The platform lead opens the item, reviews the status, and either reassigns the owner, sets a Target Resolution Date, or escalates further to the dataset business owner.

**Step 4: Resolution or acceptance**
The item moves forward through the normal resolution path. The escalation resets once a Target Resolution Date is set or Status moves to Resolution in Progress.
