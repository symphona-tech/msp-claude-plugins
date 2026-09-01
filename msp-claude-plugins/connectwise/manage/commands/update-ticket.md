---
description: Update fields on an existing ConnectWise PSA ticket
argument-hint: "<ticket_id> [status] [priority] [board] [type] [subtype] [owner] [summary]"
arguments: [ticket_id, status, priority, board, type, subtype, owner, summary]
---

# Update ConnectWise PSA Ticket

Update fields on an existing ConnectWise ticket including status, priority, board, and assignment.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold ticket update permissions
- Ticket must exist and be accessible

## Tools used

| Step | Tool |
|------|------|
| Read current state | `cw_get_ticket` |
| Resolve status | `cw_list_statuses` |
| Resolve priority | `cw_list_priorities` |
| Resolve board | `cw_list_boards` |
| Resolve owner | `cw_search_members` |
| Apply the update | `cw_update_ticket` |

## Steps

1. **Validate ticket exists**
   - `cw_get_ticket` with `id=<ticket_id>`
   - Capture the current board ID; status, type and subtype are all board-scoped

2. **Resolve field values to IDs**
   - `cw_list_statuses` with `boardId=<board_id>` — match the requested status by name
   - `cw_list_priorities` — match by name
   - `cw_list_boards` — match by name, when moving the ticket
   - `cw_search_members` with `conditions=identifier="<owner>"` — resolve the owner

   Resolve against the **destination** board when moving and changing status in
   one call: a status ID from the old board is not valid on the new one.

3. **Build the patch operations**

   `cw_update_ticket` takes JSON Patch operations, so one call carries every
   field being changed:

   ```
   cw_update_ticket
     id:         <ticket_id>
     operations: [
       {"op": "replace", "path": "status/id",   "value": <status_id>},
       {"op": "replace", "path": "priority/id", "value": <priority_id>},
       {"op": "replace", "path": "board/id",    "value": <board_id>},
       {"op": "replace", "path": "owner/id",    "value": <member_id>},
       {"op": "replace", "path": "summary",     "value": "<summary>"}
     ]
   ```

   Include only the paths the caller asked to change. Grouping related changes
   into one call keeps the update atomic — two calls can leave the ticket on a
   new board with a status from the old one.

4. **Return updated ticket summary**

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| ticket_id | integer | Yes | - | ConnectWise ticket ID to update |
| status | string | No | - | New status name |
| priority | string/int | No | - | Priority name or 1-4 |
| board | string | No | - | Target service board name |
| type | string | No | - | **Unavailable** — see below |
| subtype | string | No | - | **Unavailable** — see below |
| owner | string | No | - | Member identifier to assign |
| summary | string | No | - | New ticket summary/title |

## Examples

### Update Status

```
/update-ticket 12345 --status "In Progress"
```

### Change Priority

```
/update-ticket 12345 --priority 2
```

```
/update-ticket 12345 --priority "Priority 1 - Critical"
```

### Reassign Ticket

```
/update-ticket 12345 --owner jsmith
```

### Move to Different Board

```
/update-ticket 12345 --board "Escalations"
```

### Multiple Updates

```
/update-ticket 12345 --status "In Progress" --priority 2 --owner jsmith
```

### Update Summary

```
/update-ticket 12345 --summary "Email not working - Exchange server issue"
```

### Full Update

```
/update-ticket 12345 --status "In Progress" --priority 1 --owner jsmith --board "Escalations" --summary "Critical: Email outage affecting all users"
```

## Output

### Successful Update

```
Ticket #12345 Updated Successfully

Changes Applied:
  Status:   New -> In Progress
  Priority: Medium (3) -> High (2)
  Owner:    Unassigned -> John Smith (jsmith)

Current State:
  Company:  Acme Corporation
  Summary:  Email not working for multiple users
  Board:    Service Desk
  Status:   In Progress
  Priority: High (2)
  Owner:    John Smith (jsmith)

URL: https://na.myconnectwise.net/v4_6_release/services/system_io/Service/fv_sr100_request.rails?service_recid=12345
```

### Board Change

```
Ticket #12345 Moved Successfully

Changes Applied:
  Board:  Service Desk -> Escalations
  Status: In Progress -> New (reset due to board change)

Note: Status was reset to "New" as the previous status does not exist on the Escalations board.

Current State:
  Company:  Acme Corporation
  Summary:  Email not working for multiple users
  Board:    Escalations
  Status:   New
  Priority: High (2)

URL: https://na.myconnectwise.net/v4_6_release/services/system_io/Service/fv_sr100_request.rails?service_recid=12345
```

## Error Handling

### Ticket Not Found

```
Error: Ticket #99999 not found

The ticket may have been deleted or you may not have permission to access it.
```

### Invalid Status for Board

```
Error: Status "Completed" is not valid for board "Service Desk"

Available statuses for this board:
- New
- In Progress
- Waiting on Customer
- Waiting on Vendor
- Closed

Use one of the statuses above or move the ticket to a different board first.
```

### Invalid Board

```
Error: Service board not found: "Invalid Board"

Available boards:
- Service Desk (ID: 1)
- Managed Services (ID: 2)
- Escalations (ID: 3)
- Projects (ID: 4)
```

### Invalid Priority

```
Error: Priority not found: "Urgent"

Available priorities:
- Priority 1 - Critical (ID: 1)
- Priority 2 - High (ID: 2)
- Priority 3 - Medium (ID: 3)
- Priority 4 - Low (ID: 4)

You can also use just the number (1-4).
```

### Member Not Found

```
Error: Member not found: "invaliduser"

Did you mean one of these?
- jsmith (John Smith)
- jsmithson (Jane Smithson)
- bsmith (Bob Smith)
```

### No Changes Provided

```
Error: No changes specified

Please provide at least one field to update:
  --status     - Change ticket status
  --priority   - Change priority level
  --board      - Move to different board
  --owner      - Reassign ticket
  --summary    - Update ticket summary

Example: /update-ticket 12345 --status "In Progress"
```

### Permission Denied

```
Error: Permission denied

You do not have permission to update tickets on the "Escalations" board.
Contact your ConnectWise administrator.
```

### Closed Ticket

```
Warning: Ticket #12345 is closed

Updating a closed ticket may affect reporting.
Proceed with update? [Y/n]
```

## Not available through the tool surface

| Requested | Would need |
|-----------|------------|
| `type` — ticket type | A board type tool. Types are board-scoped and nothing in the surface lists them |
| `subtype` — ticket subtype | A board subtype tool, for the same reason |

Both were resolvable by name in the original command. **They are not now**, and
the `--type` / `--subtype` arguments cannot be honoured. Report that plainly
rather than guessing an ID: type and subtype drive board workflow and reporting
categorisation, so a wrong guess is silently miscategorised work.

A caller who knows the numeric ID can still set it, because
`cw_update_ticket` takes an arbitrary JSON Patch path — but **the ID has to come
from the operator**, not from this command.

## Related Commands