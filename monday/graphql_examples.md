# Monday.com GraphQL Examples

These are sample GraphQL patterns for interacting with Monday.com boards programmatically. They represent the integration layer between the technical framework (validation runner, monitoring tools) and the business-facing workflow boards.

These patterns follow Monday.com's v2 API structure. Board IDs, column IDs, and item IDs shown here are placeholders.

---

## Reading board data

### Query: Get all items on the Data Quality Issue Board

Retrieve open issues with their severity, owner, and status for use in an external dashboard or validation runner status check.

```graphql
query GetOpenDataQualityIssues {
  boards(ids: [BOARD_ID]) {
    items_page(
      limit: 50
      query_params: {
        rules: [
          { column_id: "status", compare_value: ["Open", "In Investigation", "Resolution in Progress"] }
        ]
      }
    ) {
      items {
        id
        name
        column_values {
          id
          text
          value
        }
        created_at
        updated_at
      }
    }
  }
}
```

---

### Query: Get migration status summary by dataset group

Pull group-level item counts for the Migration Status Board to populate a leadership summary view.

```graphql
query GetMigrationStatusByGroup {
  boards(ids: [BOARD_ID]) {
    groups {
      id
      title
      items_page(limit: 100) {
        items {
          id
          name
          column_values(ids: ["status", "validation_status", "stakeholder_signoff"]) {
            id
            text
          }
        }
      }
    }
  }
}
```

---

### Query: Get pending approvals on the Change Approval Board

Used by a deployment pipeline to check whether a change has been cleared before proceeding.

```graphql
query GetPendingApprovals($changeItemId: ID!) {
  items(ids: [$changeItemId]) {
    id
    name
    column_values(ids: ["tier_1_status", "tier_2_status", "tier_3_status", "deployment_gate"]) {
      id
      text
      value
    }
  }
}
```

---

## Writing to boards

### Mutation: Create a data quality issue from a validation failure

Called by the validation runner (`scripts/run_validation.py`) when a check fails. Creates a new item on the Data Quality Issue Board with the failure details pre-populated.

```graphql
mutation CreateDataQualityIssue(
  $boardId: ID!
  $groupId: String!
  $itemName: String!
  $columnValues: JSON!
) {
  create_item(
    board_id: $boardId
    group_id: $groupId
    item_name: $itemName
    column_values: $columnValues
  ) {
    id
    name
    created_at
  }
}
```

**Column values payload (JSON string):**

```json
{
  "detection_source": { "label": "Validation Run" },
  "affected_dataset": { "label": "Finance" },
  "severity": { "label": "High" },
  "status": { "label": "Open" },
  "affected_reports": "AR Aging Detail",
  "root_cause_category": { "label": "Source Data" },
  "description": "Record count variance 2.3% on AR Aging Detail. OBIEE source: 48,213 rows. Power BI view: 47,110 rows. Exceeds 1% threshold."
}
```

---

### Mutation: Update migration status on a report item

Called after a validation run completes to update the Migration Status Board with the result.

```graphql
mutation UpdateMigrationStatus($itemId: ID!, $columnValues: JSON!) {
  change_multiple_column_values(
    item_id: $itemId
    board_id: BOARD_ID
    column_values: $columnValues
  ) {
    id
    name
  }
}
```

**Column values payload for a passing validation:**

```json
{
  "validation_status": { "label": "Passed" },
  "validation_run_date": "2024-11-15",
  "status": { "label": "Stakeholder Review" }
}
```

---

### Mutation: Record a stakeholder sign-off

Called when the business owner confirms their sign-off through an external system or form.

```graphql
mutation RecordStakeholderSignoff($itemId: ID!, $signerName: String!) {
  change_multiple_column_values(
    item_id: $itemId
    board_id: BOARD_ID
    column_values: "{\"stakeholder_signoff\": {\"label\": \"Signed Off\"}, \"status\": {\"label\": \"Migrated\"}}"
  ) {
    id
    name
    column_values(ids: ["status", "stakeholder_signoff"]) {
      id
      text
    }
  }
}
```

---

### Mutation: Create a change approval request

Called when a platform team member initiates a change that requires the approval workflow.

```graphql
mutation CreateChangeApprovalRequest(
  $boardId: ID!
  $itemName: String!
  $columnValues: JSON!
) {
  create_item(
    board_id: $boardId
    group_id: "awaiting_tier1_approval"
    item_name: $itemName
    column_values: $columnValues
  ) {
    id
    name
    created_at
  }
}
```

**Column values payload:**

```json
{
  "change_type": { "label": "RLS Role Change" },
  "approval_tier": { "label": "Tier 3" },
  "deployment_gate": { "label": "Held" },
  "tier_1_status": { "label": "Pending" },
  "tier_2_status": { "label": "Pending" },
  "tier_3_status": { "label": "Pending" },
  "change_description": "Remove Risk_Regional_West role; reassign 14 affected users to Risk_ReadOnly pending org restructure.",
  "affected_datasets": "Risk",
  "affected_reports": "Risk Operational Summary, Credit Exposure Detail"
}
```

---

## Reading board state for deployment gating

This pattern shows how a CI/CD pipeline or deployment script can check Monday.com board state before proceeding with a production deployment.

```python
import os
import requests

MONDAY_API_KEY = os.environ["MONDAY_API_KEY"]
MONDAY_API_URL = "https://api.monday.com/v2"

def check_deployment_gate(change_item_id: str) -> bool:
    """Returns True if the change item has all approvals and gate is Released."""
    query = """
    query CheckGate($itemId: ID!) {
      items(ids: [$itemId]) {
        column_values(ids: ["deployment_gate", "tier_1_status", "tier_2_status", "tier_3_status"]) {
          id
          text
        }
      }
    }
    """
    headers = {
        "Authorization": MONDAY_API_KEY,
        "Content-Type": "application/json"
    }
    response = requests.post(
        MONDAY_API_URL,
        json={"query": query, "variables": {"itemId": change_item_id}},
        headers=headers
    )
    data = response.json()
    columns = {
        col["id"]: col["text"]
        for col in data["data"]["items"][0]["column_values"]
    }
    return columns.get("deployment_gate") == "Released"
```

---

## Webhook payload handling

Monday.com sends webhook payloads when board events occur. The sample payloads in `sample_payloads/` represent the event surface for key workflow transitions.

**Common webhook structure:**

```json
{
  "event": {
    "type": "create_pulse",
    "boardId": 1234567890,
    "itemId": 9876543210,
    "userId": 1111111,
    "triggerTime": "2024-11-15T14:32:00.000Z",
    "pulseId": 9876543210,
    "pulseName": "AR Aging Detail -- Record Count Variance 2.3%"
  }
}
```

See `sample_payloads/` for full example payloads per event type.
