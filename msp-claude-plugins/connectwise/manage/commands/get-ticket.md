---
description: Retrieve detailed ticket information from ConnectWise PSA
argument-hint: "<ticket_id> [include_notes] [include_time] [include_configs] [include_tasks]"
arguments: [ticket_id, include_notes, include_time, include_configs, include_tasks]
---

# Get ConnectWise PSA Ticket

Retrieve detailed ticket information including status, notes, and time entries.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold ticket read permissions
- Ticket must exist and be accessible

## Tools used

| Step | Tool |
|------|------|
| Read the ticket | `cw_get_ticket` |
| Read its notes | `cw_get_ticket_notes` |
| Read its time entries | `cw_search_time_entries` |

## Steps

1. **Retrieve base ticket information**
   - `cw_get_ticket` with `id=<ticket_id>`
   - Confirm the ticket is accessible before making any further call

2. **Fetch ticket notes (if include_notes=true)**
   - `cw_get_ticket_notes` with `id=<ticket_id>`

3. **Fetch time entries (if include_time=true)**
   - `cw_search_time_entries` with
     `conditions=chargeToId=<ticket_id> and chargeToType="ServiceTicket"`
   - This is a search across all time entries filtered to the ticket, not a
     ticket sub-resource, so it pages independently of the ticket read

4. **Format and return the ticket view**
   - Include whichever sections resolved
   - **Name any section that was requested and is unavailable** rather than
     returning a view that looks complete

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| ticket_id | integer | Yes | - | ConnectWise ticket ID |
| include_notes | boolean | No | true | Include ticket notes |
| include_time | boolean | No | false | Include time entries |
| include_configs | boolean | No | false | **Unavailable** — see below |
| include_tasks | boolean | No | false | **Unavailable** — see below |

## Examples

### Basic Usage

```
/get-ticket 12345
```

### With Time Entries

```
/get-ticket 12345 --include_time true
```

### Notes and Time

```
/get-ticket 12345 --include_notes true --include_time true
```

### Minimal (No Notes)

```
/get-ticket 12345 --include_notes false
```

## Output

```
================================================================================
Ticket #12345 - Email not working for multiple users
================================================================================

Company:        Acme Corporation
Contact:        John Smith (john.smith@acme.com)
Phone:          (555) 123-4567

Board:          Service Desk
Status:         In Progress
Priority:       High (2)
Type:           Incident
Subtype:        Email

Created:        2026-02-04 09:23:00 by Jane Doe
Last Updated:   2026-02-04 14:45:00

SLA Information:
  Response Due:   2026-02-04 10:23:00 (Met)
  Resolution Due: 2026-02-04 17:23:00 (2h 38m remaining)

Assigned To:    Jane Technician

--------------------------------------------------------------------------------
Description
--------------------------------------------------------------------------------
Multiple users in the sales department are unable to send or receive email.
Issue started at approximately 9am this morning after the mail server update.

================================================================================
Notes (3)
================================================================================

[INTERNAL] 2026-02-04 14:45 - Jane Technician
Identified issue with mail flow rules created during migration. Working on fix.

[EXTERNAL] 2026-02-04 11:30 - Jane Technician
Hi John, We've identified the issue and are working on a resolution. I'll update
you within the hour.

[INTERNAL] 2026-02-04 10:15 - Jane Technician
Initial triage: Confirmed mail queue is backing up. Checking mail flow rules.

================================================================================

Quick Actions:
- Update ticket: /update-ticket 12345
- Add note: /add-note 12345
- Log time: /log-time 12345
- Close ticket: /close-ticket 12345
```

### With Time Entries

```
================================================================================
Time Entries (2 entries, 2.5 hours total)
================================================================================

2026-02-04 10:00-11:30 (1.5h) - Jane Technician [Billable]
  Work Type: Remote Support
  Notes: Initial troubleshooting and triage of email issue

2026-02-04 14:00-15:00 (1.0h) - Jane Technician [Billable]
  Work Type: Remote Support
  Notes: Identified and documented root cause

Summary:
  Billable:     2.5 hours
  Non-Billable: 0.0 hours
  Total:        2.5 hours

================================================================================
```

## Error Handling

### Ticket Not Found

```
Error: Ticket #99999 not found

The ticket may have been deleted or you may not have permission to view it.
```

### Access Denied

```
Error: Access denied for ticket #12345

You do not have permission to view tickets on the "Escalations" board.
Contact your ConnectWise administrator.
```

### Rate Limited

```
Rate limited by ConnectWise API

Retrying in 5 seconds...
Successfully retrieved ticket #12345
```

### Invalid Ticket ID

```
Error: Invalid ticket ID "abc"

Ticket ID must be a numeric value.
Example: /get-ticket 12345
```

## Not available through the tool surface

Two of this command's original sections have no tool in the inventoried surface
and are **not** retrievable:

| Requested | Would need |
|-----------|------------|
| `include_configs` — configuration items linked to the ticket | A ticket-scoped configuration tool. `cw_search_configurations` searches company configurations and cannot filter by ticket |
| `include_tasks` — ticket tasks | A ticket task tool |

When either flag is passed, **say so plainly and continue with the rest**. Do
not substitute a company-wide configuration search for the ticket's linked
configurations — it answers a different question and would read as if it
answered this one.

## Related Commands