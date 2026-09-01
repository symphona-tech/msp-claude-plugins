---
description: Search for tickets in ConnectWise PSA by various criteria
argument-hint: "[query] [company] [status] [priority] [board] [assignee] [date-from] [date-to] [limit]"
arguments: [query, company, status, priority, board, assignee, date-from, date-to, limit]
---

# Search ConnectWise PSA Tickets

Search and filter tickets in ConnectWise PSA using various criteria.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold ticket read permissions

## Tools used

| Step | Tool |
|------|------|
| Resolve company | `cw_search_companies` |
| Resolve board | `cw_list_boards` |
| Resolve assignee | `cw_search_members` |
| Execute the search | `cw_search_tickets` |

## Steps

1. **Resolve names to IDs**
   - `cw_search_companies`, `cw_list_boards` and `cw_search_members` for any
     name the caller supplied instead of an ID
   - Map status and priority text to conditions using the tables below

2. **Build the conditions string**

   `cw_search_tickets` takes ConnectWise conditions syntax directly, so the
   query is composed rather than URL-built:

   ```
   cw_search_tickets
     conditions: "company/id=12345 and closedFlag=false"
     orderBy:    "priority/id asc, dateEntered desc"
     pageSize:   <limit>
   ```

   Every argument is optional. With no `conditions` the server returns the
   default page, so always pass a filter unless the caller asked for everything.

3. **Page through results if needed**
   - `page` and `pageSize` are tool arguments; `pageSize` maxes at 1000
   - Request another page only when the caller asked for more than one returned

4. **Format and return results**
   - Display ticket list with key details
   - Include quick actions for each ticket

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| query | string | No | - | Free text search in summary |
| company | string/int | No | - | Company filter |
| status | string | No | open | open/closed/all or specific status |
| priority | string/int | No | - | critical/high/medium/low or 1-4 |
| board | string/int | No | - | Service board filter |
| assignee | string | No | - | Assigned member filter |
| date-from | date | No | - | Created on/after date |
| date-to | date | No | - | Created before date |
| limit | int | No | 25 | Max results (1-1000) |

## Examples

### Search by Text

```
/search-tickets "email not working"
```

### Filter by Company

```
/search-tickets --company "Acme Corp"
```

### High Priority Open Tickets

```
/search-tickets --priority high --status open
```

### My Assigned Tickets

```
/search-tickets --assignee "me"
```

### Combined Filters

```
/search-tickets --company "Acme" --status open --board "Service Desk" --limit 10
```

### Search by Ticket ID

```
/search-tickets "54321"
```

### Date Range Search

```
/search-tickets --date-from "2024-02-01" --date-to "2024-02-15" --status all
```

### Critical Priority Only

```
/search-tickets --priority 1 --status open
```

## Output

```
Found 3 tickets matching criteria

+----------+----------------------------------+------------------+----------+-------------+
| ID       | Summary                          | Company          | Priority | Status      |
+----------+----------------------------------+------------------+----------+-------------+
| 54321    | Email not working                | Acme Corporation | High     | In Progress |
| 54318    | Outlook crashes on startup       | Acme Corporation | Medium   | New         |
| 54305    | Cannot send attachments > 10MB   | Acme Corporation | Low      | Waiting     |
+----------+----------------------------------+------------------+----------+-------------+

Quick Actions:
- View ticket: /get-ticket <id>
- Update ticket: /update-ticket <id>
- Add note: /add-note <id>
```

### Detailed View

```
/search-tickets --company "Acme" --limit 2
```

```
Found 2 tickets for Acme Corporation

================================================================================
#54321 - Email not working
================================================================================
Company:    Acme Corporation
Contact:    John Smith (john.smith@acme.com)
Priority:   High (2)
Status:     In Progress
Board:      Service Desk
Assignee:   Jane Technician
Created:    2024-02-15 09:23:00
Updated:    2024-02-15 10:45:00
SLA Due:    2024-02-15 11:23:00 (38 min remaining)

Description:
Multiple users unable to send or receive email since 9am.
Affects sales team primarily.

(Notes are a separate read: `cw_get_ticket_notes` per ticket, or `/get-ticket <id>`.)
================================================================================
```

## Filter Reference

### Status Values

| Text | Condition Generated |
|------|---------------------|
| open | `closedFlag=false` |
| closed | `closedFlag=true` |
| all | No status filter |
| new | `status/name="New"` |
| in-progress | `status/name="In Progress"` |
| waiting | `status/name contains "Waiting"` |
| completed | `status/name="Completed"` |

### Priority Values

| Text | Priority ID | Condition |
|------|-------------|-----------|
| critical | 1 | `priority/id=1` |
| medium | 3 | `priority/id=3` |
| high | 2 | `priority/id=2` |
| low | 4 | `priority/id=4` |

### Date Formats

- `YYYY-MM-DD` - Date only (2024-02-15)
- `YYYY-MM-DDTHH:MM:SS` - Date and time (2024-02-15T09:00:00)

## Conditions Examples

These are the strings passed as the `conditions` argument.

**Open tickets for company:**
```
company/id=12345 and closedFlag=false
```

**High priority open tickets:**
```
priority/id<=2 and closedFlag=false
```

**Text search:**
```
summary contains "email" and closedFlag=false
```

**Date range:**
```
dateEntered>=[2024-02-01] and dateEntered<[2024-02-15]
```

**Assigned to specific member:**
```
resources contains "jsmith" and closedFlag=false
```

See the `connectwise-psa-api-patterns` skill for the full conditions grammar.

## Error Handling

### No Results

```
No tickets found matching criteria

Suggestions:
- Broaden your search (remove filters)
- Check spelling of company/board names
- Try --status all to include closed tickets
- Verify date format is YYYY-MM-DD
```

### Invalid Company

```
Error: Company not found: "Acm"

Did you mean?
- Acme Corporation (ID: 12345)
- Acme Industries (ID: 12346)
```

### Invalid Board

```
Error: Service board not found: "Invalid"

Available boards:
- Service Desk (ID: 1)
- Managed Services (ID: 2)
- Projects (ID: 3)
```

### Rate Limiting

```
Rate limited by ConnectWise API

Retrying in 5 seconds...
```

### Too Many Results

```
Found 1,247 tickets matching criteria (showing first 25)

Use --limit to increase results or add filters to narrow search.
```

## Ordering and Fields

`orderBy` takes the same syntax as the API: `priority/id asc, dateEntered desc`
puts critical first and newest first within a priority.

Field selection is **not** a `cw_search_tickets` argument — the tool returns the
server's default projection. Filter the result rather than asking for a subset.

## Related Commands

- `/create-ticket` - Create new ticket
- `/get-ticket` - View full ticket details
- `/update-ticket` - Modify ticket
- `/add-note` - Add note to ticket
- `/log-time` - Log time against ticket
