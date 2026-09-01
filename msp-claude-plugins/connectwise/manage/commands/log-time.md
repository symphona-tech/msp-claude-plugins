---
description: Log a time entry against a ConnectWise PSA ticket
argument-hint: "<ticket_id> <time_start> [time_end] [actual_hours] [notes] [billable] [work_type] [work_role]"
arguments: [ticket_id, time_start, time_end, actual_hours, notes, billable, work_type, work_role]
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
     from the caller — the server authenticates as one API member, which is not
     necessarily the technician the time belongs to

3. **Calculate time values**
   - `timeStart` is required; supply `timeEnd` or `actualHours`
   - Both are ISO 8601

4. **Check agreement coverage (if billable)**
   - `cw_search_agreements` with
     `conditions=company/id=<company_id> and cancelledFlag=false`
   - Warn when the entry would exceed remaining covered hours

5. **Create the time entry**

   ```
   cw_create_time_entry
     chargeToType:  "ServiceTicket"
     chargeToId:    <ticket_id>
     memberId:      <member_id>
     timeStart:     "<ISO 8601>"
     timeEnd:       "<ISO 8601>"
     notes:         "<work notes>"
     internalNotes: "<internal notes>"
   ```

   `chargeToType`, `chargeToId`, `memberId` and `timeStart` are all required.
   **A time entry against a ticket lands on the customer's next invoice** — treat
   it as a billing action and confirm before creating one.

6. **Return confirmation with totals**

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| ticket_id | integer | Yes | - | Ticket ID to log time against |
| time_start | datetime | Yes | - | Start time (YYYY-MM-DD HH:MM or "now") |
| time_end | datetime | No* | - | End time |
| actual_hours | decimal | No* | - | Hours worked (alternative to time_end) |
| notes | string | No | - | Work description |
| billable | string | No | Billable | Billable, DoNotBill, NoCharge |
| work_type | string | No | - | **Unavailable** — see below |
| work_role | string | No | - | **Unavailable** — see below |

*Either `time_end` or `actual_hours` is required

## Examples

### Using Actual Hours

```
/log-time 12345 "2026-02-04 09:00" --actual_hours 1.5 --notes "Troubleshot network connectivity"
```

### Using Start/End Times

```
/log-time 12345 "2026-02-04 10:00" "2026-02-04 11:30" --notes "Remote support session"
```

### Log Time Starting Now

```
/log-time 12345 "now" --actual_hours 0.5 --notes "Quick phone support"
```

### Non-Billable Time

```
/log-time 12345 "2026-02-04 14:00" --actual_hours 0.5 --billable DoNotBill --notes "Internal documentation"
```

### No-Charge Time

```
/log-time 12345 "2026-02-04 15:00" --actual_hours 0.25 --billable NoCharge --notes "Courtesy follow-up"
```

### Full Example

```
/log-time 12345 "2026-02-04 09:00" "2026-02-04 11:30" --notes "Server migration assistance" --billable Billable
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
  Status:     Billable
  Work Type:  Remote Support
  Work Role:  Engineer
  Rate:       $150.00/hour
  Amount:     $225.00

Notes:
"Troubleshot network connectivity issues. Identified DNS misconfiguration."

Ticket Time Summary:
  This Entry:  1.5 hours
  Previous:    2.0 hours
  Total:       3.5 hours (3.5 billable)
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

### Invalid Billable Option

```
Error: Invalid billable option "Bill"

Valid options:
- Billable    - Time will be billed to customer
- DoNotBill   - Time tracked but not billed
- NoCharge    - Time marked as no charge

Example: /log-time 12345 "now" --actual_hours 1 --billable DoNotBill
```

### No Agreement Warning

```
Warning: No active agreement for Acme Corporation

Time will be billed at Time & Materials rates.
Work Type: Remote Support
Rate: $175.00/hour
Estimated: $262.50

Proceed? [Y/n]
Mark as non-billable? [n]
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
| `work_type` — resolve a work type by name | A work type tool |
| `work_role` — resolve a work role by name | A work role tool |

Neither can be resolved by name, and `cw_create_time_entry`'s `workTypeId` and
`workRoleId` therefore have no lookup behind them. When the caller names one,
**say it cannot be resolved and create the entry without it**, letting the
agreement and board defaults apply — rather than guessing an ID.

This matters more than the other gaps in this plugin: the work role is what
sets the **billing rate**. A guessed `workRoleId` bills the customer at the
wrong rate, and nothing downstream would flag it.

## Related Commands