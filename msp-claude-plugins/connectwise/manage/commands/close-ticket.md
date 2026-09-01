---
description: Close a ConnectWise PSA ticket with resolution notes
argument-hint: "<ticket_id> <resolution> [status] [member] [time_minutes] [time_notes]"
arguments: [ticket_id, resolution, status, member, time_minutes, time_notes]
---

# Close ConnectWise PSA Ticket

Close a ConnectWise ticket with resolution notes and optional time entry.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold ticket update permissions
- Ticket must exist and be accessible
- A valid closed status must exist on the ticket's board

## Tools used

**There is no close tool.** Closure is composed from five:

| Step | Tool |
|------|------|
| Read current state | `cw_get_ticket` |
| Resolve a closed status | `cw_list_statuses` |
| Add the resolution note | `cw_add_ticket_note` |
| Resolve the member | `cw_search_members` |
| Log final time (optional) | `cw_create_time_entry` |
| Move status | `cw_update_ticket` |

Closing is not a bookkeeping change: it stops the SLA clock and fires whatever
close notification and satisfaction survey the board is wired to. Confirm with
the operator before either `cw_update_ticket` step.

## Steps

1. **Validate ticket exists and get current state**
   - `cw_get_ticket` with `id=<ticket_id>`
   - Capture the current board ID and the current status
   - Check if the ticket is already closed

2. **Resolve the closed status**
   - `cw_list_statuses` with `boardId=<board_id>` and `conditions=closedStatus=true`
   - If a status parameter was provided, confirm it appears in that result
   - Otherwise use the board's default closed status

3. **Add the resolution note**

   ```
   cw_add_ticket_note
     id:                  <ticket_id>
     text:                "<resolution>"
     resolutionFlag:      true
     customerUpdatedFlag: true
   ```

4. **Log a time entry (if time_minutes provided)**

   Resolve the member first — `cw_create_time_entry` requires `memberId` and
   will not infer it. The server authenticates as one API member, which is not
   the technician the time belongs to.

   ```
   cw_search_members  conditions: "identifier=\"<member>\""

   cw_create_time_entry
     chargeToType: "ServiceTicket"
     chargeToId:   <ticket_id>
     memberId:     <resolved_member_id>
     timeStart:    "<ISO 8601 start>"
     timeEnd:      "<ISO 8601 end>"
     notes:        "<time_notes or resolution>"
   ```

   **If `member` was not supplied, stop and ask for it.** Do not fall back to
   the API member the server authenticates as — that would attribute a
   technician's billable time to a shared service account.

5. **Move the ticket through `Completed` before `Closed`**

   A ticket **must pass through `Completed` before `Closed`**. Patching straight
   to a closed status fails with an error whose message reads like a permissions
   failure, which is how this goes wrong quietly.

   If the current status is not already `Completed`, patch there first:

   ```
   cw_update_ticket
     id:         <ticket_id>
     operations: [{"op": "replace", "path": "status/id", "value": <completed_status_id>}]
   ```

   Resolve `<completed_status_id>` from the same `cw_list_statuses` call — a
   `Completed` status is **not** a closed status, so look it up without the
   `closedStatus=true` filter.

6. **Set the closed status and the resolution**

   ```
   cw_update_ticket
     id:         <ticket_id>
     operations: [
       {"op": "replace", "path": "status/id",  "value": <closed_status_id>},
       {"op": "replace", "path": "resolution", "value": "<resolution>"}
     ]
   ```

7. **Return closure confirmation**

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| ticket_id | integer | Yes | - | ConnectWise ticket ID to close |
| resolution | string | Yes | - | Resolution notes |
| status | string | No | Board default | Closed status name |
| member | string | Conditional | - | Member identifier the time entry belongs to. **Required when `time_minutes` is given** |
| time_minutes | integer | No | - | Final time entry in minutes |
| time_notes | string | No | - | Notes for time entry |

**There is no `billable` parameter.** `cw_create_time_entry` exposes no billing
argument, so this command cannot mark an entry non-billable — see below.

## Examples

### Basic Closure

```
/close-ticket 12345 "Password reset completed and verified with user"
```

### With Time Entry

```
/close-ticket 12345 "Server patched and rebooted successfully" --member jtech --time_minutes 30 --time_notes "Applied security patches"
```

### With an Explicit Member

```
/close-ticket 12345 "Hardware replaced under warranty" --member jtech --time_minutes 60
```

### Specific Closed Status

```
/close-ticket 12345 "Issue resolved" --status "Completed"
```

### Full Closure with All Options

```
/close-ticket 12345 "Configured new email account and tested send/receive functionality" --status "Completed" --member jtech --time_minutes 45 --time_notes "Email configuration and testing"
```

## Output

### Basic Closure

```
Ticket #12345 Closed Successfully

Resolution:
"Password reset completed and verified with user"

Final State:
  Company:    Acme Corporation
  Summary:    Password reset request
  Board:      Service Desk
  Status:     Closed
  Priority:   Medium (3)

Closed By:   Jane Technician
Closed At:   2026-02-04 15:30:00

Total Time:  1.5 hours

URL: https://na.myconnectwise.net/v4_6_release/services/system_io/Service/fv_sr100_request.rails?service_recid=12345
```

### With Time Entry

```
Ticket #12345 Closed Successfully

Resolution:
"Server patched and rebooted successfully"

Time Entry Added:
  Duration:   30 minutes (0.5 hours)
  Notes:      Applied security patches
  Work Type:  Remote Support

Final State:
  Company:    Acme Corporation
  Summary:    Server maintenance required
  Board:      Managed Services
  Status:     Completed
  Priority:   Medium (3)

Closed By:   Jane Technician
Closed At:   2026-02-04 16:00:00

Time Summary:
  Previous:   2.0 hours
  This Entry: 0.5 hours
  Total:      2.5 hours

URL: https://na.myconnectwise.net/v4_6_release/services/system_io/Service/fv_sr100_request.rails?service_recid=12345
```

## Error Handling

### Ticket Not Found

```
Error: Ticket #99999 not found

The ticket may have been deleted or you may not have permission to access it.
```

### Ticket Already Closed

```
Warning: Ticket #12345 is already closed

Current Status: Closed
Closed On:      2026-02-04 10:00:00
Resolution:     "Issue resolved by restarting service"

Reopen and close again? [Y/n]
Update resolution only? [u]
Cancel? [c]
```

### Empty Resolution

```
Error: Resolution notes are required

Please provide resolution details:
/close-ticket 12345 "Describe how the issue was resolved"
```

### Invalid Closed Status

```
Error: Status "In Progress" is not a closed status

Available closed statuses for this board:
- Closed
- Completed
- Cancelled

Example: /close-ticket 12345 "Resolution" --status "Completed"
```

### No Closed Status on Board

```
Error: No closed status found on board "Projects"

This board may not support ticket closure. Contact your ConnectWise administrator
to add a closed status or move the ticket to a different board first.
```

### Time Entry Validation

```
Error: Invalid time entry

time_minutes must be a positive number.
Example: /close-ticket 12345 "Resolution" --time_minutes 30
```

### Agreement Validation

```
Warning: Time entry may not be covered by agreement

Company: Acme Corporation
Agreement: Managed Services (Block Hours)
Remaining Hours: 0.25 hours
Time to Log: 0.5 hours

This will exceed the agreement by 0.25 hours (billed as overage).

Proceed? [Y/n]
Cancel? [c]
```

### Permission Denied

```
Error: Permission denied

The API member does not have permission to close tickets on the "Escalations" board.
Contact your ConnectWise administrator.
```

### SLA Warning

```
Warning: Closing ticket #12345 past SLA

Resolution SLA Due:  2026-02-04 14:00:00
Current Time:        2026-02-04 16:30:00
SLA Breach:          2 hours 30 minutes

Proceed with closure? [Y/n]
```

### Partial Failure

Closure is four tool calls, not one transaction. If `cw_update_ticket` fails
after the note and the time entry landed, **say exactly which steps completed**
— the ticket is open with a resolution note attached, and a retry that repeats
every step would double-log the time.

## Not available through the tool surface

| Requested | Would need |
|-----------|------------|
| Marking the final time entry non-billable | A billing argument on `cw_create_time_entry` |

`cw_create_time_entry` accepts `chargeToType`, `chargeToId`, `memberId`,
`timeStart`, `timeEnd`, `actualHours`, `notes`, `internalNotes`, `workTypeId`
and `workRoleId` — **and no billing argument at all.** A time entry read back
from `cw_search_time_entries` carries `billableOption`, so the field exists on
the record; it simply cannot be set through this surface.

**An entry created here therefore takes the server's default billing
treatment, which in most configurations is billable.** If the work must not be
billed, do not log it through this command — record the time in the PSA
directly. Never create the entry and describe it as non-billable.

## Related Commands