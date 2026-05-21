# Monday.com Automation Rules

This document defines the automation and notification rule patterns for the five-board operating model. Rules are written in plain language following Monday.com's trigger-action structure.

## Report Intake Board

**Rule: Notify platform team on new submission**
- Trigger: Item is created in any group
- Condition: Intake Review Status is empty
- Action: Send notification to Platform Team channel with item name, request type, source dataset, and priority

**Rule: Set submission date on creation**
- Trigger: Item is created
- Action: Set Submission Date to today

**Rule: Warn on triage SLA breach**
- Trigger: Time arrives at 3 business days after Submission Date
- Condition: Intake Review Status is Pending or empty
- Action: Send notification to Reviewer (if assigned) or Platform Lead: "Intake SLA: [item name] has been in triage for 3 days without a decision."

**Rule: Move to backlog and create migration item on approval**
- Trigger: Intake Review Status changes to Approved
- Action: Move item to Approved - In Backlog group; create linked item on Migration Status Board with Item Name, Target Dataset, Owner, and Priority copied; set linked item's Migration Status to Pending

**Rule: Notify requestor on rejection**
- Trigger: Intake Review Status changes to Rejected
- Condition: Review Notes is not empty
- Action: Send notification to Requestor with review notes text

---

## Migration Status Board

**Rule: Notify owner on status change to Blocked**
- Trigger: Migration Status changes to Blocked
- Condition: Blocker Description is not empty
- Action: Send notification to Owner and Platform Lead with item name and blocker description

**Rule: Alert on stale In Progress items**
- Trigger: Time arrives at 14 days after status was set to In Progress
- Condition: Migration Status is still In Progress
- Action: Send notification to Owner: "[Report name] has been In Progress for 14 days. Please update status or note blockers."

**Rule: Notify business owner when status reaches Stakeholder Review**
- Trigger: Migration Status changes to Stakeholder Review
- Action: Send notification to Signoff Owner: "[Report name] is ready for your review and sign-off. Please confirm the numbers match your expectations."

**Rule: Update status to Migrated on sign-off**
- Trigger: Stakeholder Signoff changes to Signed Off
- Action: Set Migration Status to Migrated; set Go Live Date to today if empty

**Rule: Weekly leadership rollup**
- Trigger: Every Monday at 8:00 AM
- Action: Send summary notification to Platform Lead with count of items by Migration Status group

---

## Data Quality Issue Board

**Rule: Set detection date on creation**
- Trigger: Item is created
- Action: Set Detection Date to today

**Rule: Notify dataset owner immediately on Critical**
- Trigger: Severity is set to Critical
- Action: Send immediate notification to Owner and Platform Lead: "Critical data quality issue opened: [item name]. Affected dataset: [dataset]. Detection source: [source]. Immediate attention required."

**Rule: Notify dataset owner on High**
- Trigger: Severity is set to High
- Action: Send notification to Owner: "High-severity data quality issue opened: [item name]. Please begin investigation and update status within 24 hours."

**Rule: Escalate stale High and Critical issues**
- Trigger: Every day at 7:00 AM
- Condition: Severity is High or Critical AND Status is not Resolved or Accepted AND Detection Date is more than 14 days ago
- Action: Send notification to Platform Lead with item name, owner, severity, and days open

**Rule: Require validation evidence before Resolved**
- Trigger: Status changes to Resolved
- Condition: Validation Evidence is empty
- Action: Set Status back to In Investigation; send notification to Owner: "Resolved status requires validation evidence. Please document the passing validation run or acceptance record before closing."

**Rule: Require business owner signoff for Accepted**
- Trigger: Status changes to Accepted
- Condition: Business Owner Signoff is not Signed Off
- Action: Set Status back to In Investigation; send notification to Owner: "Accepted status requires business owner sign-off. Please obtain sign-off before closing."

---

## Change Approval Board

**Rule: Hold deployment gate on creation**
- Trigger: Item is created
- Action: Set Deployment Gate to Held

**Rule: Release gate when all required approvals are recorded**
- Trigger: Tier 1 Status changes to Approved
- Condition: Tier 2 Status is Approved or Not Required AND Tier 3 Status is Approved or Not Required
- Action: Set Deployment Gate to Released; notify Requestor and Owner: "All required approvals recorded. [Change name] is cleared for deployment."
- (Mirror rules exist for changes to Tier 2 and Tier 3 Status fields)

**Rule: Notify approver on Tier 1 assignment**
- Trigger: Tier 1 Approver is assigned
- Action: Send notification to Tier 1 Approver: "Your approval is required for: [change name]. Please review the change description and downstream impact assessment."

**Rule: Notify approver on Tier 2 assignment (when not Not Required)**
- Trigger: Tier 2 Approver is assigned AND Tier 2 Status is not Not Required
- Action: Send notification to Tier 2 Approver: "Your approval is required for: [change name]. Approval tier: 2. Please review."

**Rule: Notify approver on Tier 3 assignment (when not Not Required)**
- Trigger: Tier 3 Approver is assigned AND Tier 3 Status is not Not Required
- Action: Send notification to Tier 3 Approver: "Your approval is required for: [change name]. This change affects row-level security. Please review access impact."

**Rule: Escalate stale pending approvals**
- Trigger: Every day at 8:00 AM
- Condition: Deployment Gate is Held AND item was created more than 5 days ago AND any approval status is Pending
- Action: Send notification to Platform Lead with pending approver name and change name

**Rule: Require post-deploy validation before marking Deployed**
- Trigger: Actual Deploy Date is set
- Action: Set Post-Deploy Validation to Not Started; notify Owner: "Deployment recorded. Please run post-deployment validation and update the validation status."

---

## Platform Operations Board

**Rule: Set open date on creation**
- Trigger: Item is created
- Action: Set Open Date to today

**Rule: Notify platform team on Critical or High priority creation**
- Trigger: Item is created AND Priority is Critical or High
- Action: Send notification to Platform Team channel with item name, type, and priority

**Rule: Notify owner on assignment**
- Trigger: Owner is assigned
- Action: Send notification to Owner: "You have been assigned: [item name]. Type: [item type]. Priority: [priority]."

**Rule: Escalate unowned Critical items**
- Trigger: Time arrives at 1 hour after item creation
- Condition: Priority is Critical AND Owner is empty
- Action: Send notification to Platform Lead: "Critical platform operations item has no owner after 1 hour: [item name]."

**Rule: Request manager approval for access requests**
- Trigger: Item Type is set to Access Request AND Manager is assigned
- Action: Set Manager Approval to Pending; send notification to Manager: "Access request requires your approval: [item name]. Requestor: [requestor]. Please review and approve or reject."

**Rule: Set close date on resolution**
- Trigger: Status changes to Resolved or Closed
- Action: Set Close Date to today
