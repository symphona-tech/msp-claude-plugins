---
description: Log a time entry against a ConnectWise PSA ticket
argument-hint: "<ticket_id> <member> <time_start> [time_end] [actual_hours] [notes]"
arguments: [ticket_id, member, time_start, time_end, actual_hours, notes]
---

# Log Time to ConnectWise PSA Ticket

Log a time entry against a ConnectWise ticket.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold time entry permissions
- Ticket must exist and be accessible

## Tools used

| Step | Tool |
|------|------|
| Validate the ticket | `cw_get_ticket` |
| Resolve the member | `cw_search_members` |
| Check agreement coverage | `cw_search_agreements` |
| Create the entry | `cw_create_time_entry` |

## Steps

1. **Validate ticket exists**
   - `cw_get_ticket` with `id=<ticket_id>`
   - Capture the company ID for the agreement check

2. **Resolve the member**
   - `cw_search_members` with `conditions=identifier="<member>"`
   - `memberId` is **required** by `cw_create_time_entry` and is not inferred
     from the caller. The server authenticates as one API member, which is not
     the technician the time belongs to.
   - **If `member` was not supplied, stop and ask.** Never fall back to the API
     member the server authenticates as: that attributes billable work to a
     shared service account and misstates who did it.

3. **Calculate time values**
   - `timeStart` is required; supply `timeEnd` or `actualHours`
   - Both are ISO 8601

4. **Check agreement coverage**
   - `cw_search_agreements` with
     `conditions=company/id=<company_id> and cancelledFlag=false and (endDate >= [<today>] or endDate = null)`
   - Warn when the entry would exceed remaining covered hours

5. **Create the time entry**

   ```
   cw_create_time_entry
     chargeToType:  "ServiceTicket"
     chargeToId:    <ticket_id>
     memberId:      <resolved_member_id>
     timeStart:     "<ISO 8601>"
     timeEnd:       "<ISO 8601>"
     notes:         "<work notes>"
     internalNotes: "<internal notes>"
   ```

   `chargeToType`, `chargeToId`, `memberId` and `timeStart` are all required.
   **A time entry against a ticket lands on the customer's next invoice** —
   treat it as a billing action and confirm before creating one.

6. **Return confirmation with totals**

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| ticket_id | integer | Yes | - | ConnectWise ticket ID |
| member | string | Yes | - | Member identifier the time belongs to |
| time_start | string | Yes | - | Start time (ISO 8601, or "now") |
| time_end | string | No | - | End time (ISO 8601) |
| actual_hours | number | No | - | Hours worked, as an alternative to time_end |
| notes | string | No | - | Work notes, visible to the customer |

**There is no `billable` parameter, and no work type or work role.** None of the
three can be expressed through `cw_create_time_entry` — see below.

## Examples

### Using Actual Hours

```
/log-time 12345 jtech "2026-02-04 09:00" --actual_hours 1.5 --notes "Troubleshot network connectivity"
```

### Using Start/End Times

```
/log-time 12345 jtech "2026-02-04 10:00" "2026-02-04 11:30" --notes "Remote support session"
```

### Log Time Starting Now

```
/log-time 12345 jtech "now" --actual_hours 0.5 --notes "Quick phone support"
```

### Full Example

```
/log-time 12345 jtech "2026-02-04 09:00" "2026-02-04 11:30" --notes "Server migration assistance"
```

## Output

### Successful Time Entry

```
Time Entry Logged Successfully

Entry ID:     67890
Ticket:       #12345 - Email not working
Company:      Acme Corporation

Time Details:
  Start:      2026-02-04 09:00
  End:        2026-02-04 10:30
  Duration:   1.5 hours

Billing:
  Work Type:  Remote Support
  Work Role:  Engineer
  Rate:       $150.00/hour
  Amount:     $225.00

Notes:
"Troubleshot network connectivity issues. Identified DNS misconfiguration."

Ticket Time Summary:
  This Entry:  1.5 hours
  Previous:    2.0 hours
  Total:       3.5 hours
```

### With Agreement Info

```
Time Entry Logged Successfully

Entry ID:     67891
Ticket:       #12345 - Email not working
Company:      Acme Corporation

Time Details:
  Start:      2026-02-04 14:00
  End:        2026-02-04 15:00
  Duration:   1.0 hours

Agreement Coverage:
  Agreement:  Managed Services - Block Hours
  Used:       1.0 hours from block
  Remaining:  15.5 hours (after this entry)
  Expires:    2026-03-01

Notes:
"Applied configuration changes and verified resolution."
```

## Error Handling

### Ticket Not Found

```
Error: Ticket #99999 not found

The ticket may have been deleted or you may not have permission to access it.
```

### Invalid Time Format

```
Error: Invalid time format "02/04/2026"

Supported formats:
- YYYY-MM-DD HH:MM (e.g., 2026-02-04 09:00)
- YYYY-MM-DDTHH:MM:SS (e.g., 2026-02-04T09:00:00)
- "now" (current time)

Example: /log-time 12345 "2026-02-04 09:00" --actual_hours 1.5
```

### Missing Duration

```
Error: Either time_end or actual_hours is required

Examples:
  /log-time 12345 "2026-02-04 09:00" "2026-02-04 10:30"
  /log-time 12345 "2026-02-04 09:00" --actual_hours 1.5
```

### End Before Start

```
Error: End time must be after start time

Start: 2026-02-04 10:00
End:   2026-02-04 09:00

Please check your time values.
```

### Future Time Warning

```
Warning: Time entry is in the future

Start: 2026-02-05 09:00 (tomorrow)

Log future time entry? [Y/n]
```

### No Agreement Warning

```
Warning: No active agreement for Acme Corporation

Time will be billed at Time & Materials rates.
Work Type: Remote Support
Rate: $175.00/hour
Estimated: $262.50

Proceed? [Y/n]
```

### Agreement Hours Exceeded

```
Warning: Agreement hours will be exceeded

Agreement: Managed Services - Block Hours
Remaining: 0.5 hours
Logging:   1.5 hours
Overage:   1.0 hours (billed at T&M rates)

Proceed? [Y/n]
Split entry? [s] (0.5h covered, 1.0h T&M)
Mark as non-billable? [n]
```

### Closed Ticket Warning

```
Warning: Ticket #12345 is closed

Adding time to closed tickets may affect reporting.
Log time anyway? [Y/n]
```

### Permission Denied

```
Error: Permission denied

You do not have permission to log time against tickets on the "Escalations" board.
Contact your ConnectWise administrator.
```

### Overlapping Time Entry

```
Warning: Overlapping time entry detected

Existing entry: 2026-02-04 09:00 - 10:30 (Ticket #12340)
New entry:      2026-02-04 09:30 - 11:00 (Ticket #12345)

Create overlapping entry? [Y/n]
Adjust start time to 10:30? [a]
Cancel? [c]
```

## Not available through the tool surface

| Requested | Would need |
|-----------|------------|
| Marking an entry non-billable (`DoNotBill`, `NoCharge`) | A billing argument on `cw_create_time_entry` |
| `work_type` — resolve a work type by name | A work type tool |
| `work_role` — resolve a work role by name | A work role tool |

`cw_create_time_entry` accepts `chargeToType`, `chargeToId`, `memberId`,
`timeStart`, `timeEnd`, `actualHours`, `notes`, `internalNotes`, `workTypeId`
and `workRoleId` — **and no billing argument at all.** An entry read back
through `cw_search_time_entries` carries `billableOption`, so the field exists
on the record and simply cannot be set from here.

**An entry created by this command takes the server's default billing
treatment, which in most configurations is billable.** If work must not be
billed, **do not log it through this command** — record it in the PSA directly.
Never create an entry and then describe it as non-billable.

Work types and roles have no lookup tool, so `workTypeId` and `workRoleId`
cannot be resolved from a name. This matters more than the other gaps here: the
work role sets the **billing rate**, so a guessed ID bills the customer at the
wrong rate and nothing downstream would flag it. Create the entry without them
and let the agreement and board defaults apply.

## Related Commands