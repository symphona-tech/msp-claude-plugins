---
description: Create a new service ticket in ConnectWise PSA
argument-hint: "<company> <summary> [description] [board] [priority] [contact] [status]"
arguments: [company, summary, description, board, priority, contact, status]
---

# Create ConnectWise PSA Ticket

Create a new service ticket in ConnectWise PSA with specified details.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- Company must exist in ConnectWise PSA
- The server's API member must hold ticket creation permissions

## Tools used

| Step | Tool |
|------|------|
| Resolve company | `cw_search_companies` |
| Check for duplicates | `cw_search_tickets` |
| Resolve board | `cw_list_boards` |
| Resolve contact | `cw_search_contacts` |
| Resolve priority | `cw_list_priorities` |
| Resolve status | `cw_list_statuses` |
| Check agreement coverage | `cw_search_agreements` |
| Create the ticket | `cw_create_ticket` |

## Steps

1. **Validate company exists**
   - If numeric, use as company ID directly
   - Otherwise call `cw_search_companies` with `conditions=name contains "<company>" or identifier="<company>"`
   - Suggest similar companies if no exact match found

2. **Check for duplicate tickets**
   - `cw_search_tickets` with `conditions=company/id=<id> and closedFlag=false`
   - Warn if similar summaries found in last 24 hours

3. **Resolve service board**
   - `cw_list_boards` and match by name or ID
   - If not specified, use company default or first available board
   - Validate board exists and is active

4. **Resolve optional fields**
   - `cw_search_contacts` with `conditions=company/id=<id>` to look up a contact by name or email
   - Validate contact belongs to the company
   - `cw_list_priorities` to map priority text to ID (Critical=1, High=2, Medium=3, Low=4)
   - `cw_list_statuses` with `boardId=<id>` to resolve the status, or use the board default "New"

5. **Check agreement coverage**
   - `cw_search_agreements` with `conditions=company/id=<id> and cancelledFlag=false`
   - Warn if no active agreement (may be T&M billing)

6. **Create the ticket**

   `cw_create_ticket` takes resolved numeric IDs rather than nested objects:

   ```
   cw_create_ticket
     summary:            "<summary>"
     boardId:            <resolved_board_id>
     companyId:          <resolved_company_id>
     contactId:          <resolved_contact_id>
     priorityId:         <resolved_priority_id>
     statusId:           <resolved_status_id>
     initialDescription: "<description>"
   ```

   Only `summary` is required. Omit any field that did not resolve rather than
   guessing an ID — a wrong `boardId` files the ticket on the wrong queue and
   silently changes which SLA applies.

7. **Return ticket details**
   - Ticket ID
   - Ticket summary
   - Direct URL to ticket in ConnectWise PSA

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| company | string/int | Yes | - | Company name, identifier, or ID |
| summary | string | Yes | - | Brief summary (max 100 chars) |
| description | string | No | - | Detailed issue description |
| board | string/int | No | Company default | Service board name or ID |
| priority | int/string | No | 3 (Medium) | 1=Critical, 2=High, 3=Medium, 4=Low |
| contact | string | No | - | Contact name or email |
| status | string/int | No | New | Initial ticket status |

## Examples

### Basic Usage

```
/create-ticket "Acme Corp" "Email not working"
```

### With Full Details

```
/create-ticket "Acme Corp" "Email not working" --description "Multiple users unable to send/receive since 9am" --priority 2 --contact "john.smith@acme.com" --board "Service Desk"
```

### Using Company Identifier

```
/create-ticket "ACME" "Server offline" --priority 1
```

### Using Company ID

```
/create-ticket 12345 "Network slow" --priority 3 --board "Managed Services"
```

## Output

```
Ticket Created Successfully

Ticket ID: 54321
Summary: Email not working
Company: Acme Corporation
Priority: High (2)
Status: New
Board: Service Desk
Contact: John Smith (john.smith@acme.com)
Agreement: Managed Services Agreement (Active)

URL: https://na.myconnectwise.net/v4_6_release/services/system_io/Service/fv_sr100_request.rails?service_recid=54321
```

## Error Handling

### Company Not Found

```
Error: Company not found: "Acm"

Did you mean one of these?
- Acme Corporation (Identifier: ACME, ID: 12345)
- Acme Industries (Identifier: ACMEIND, ID: 12346)
- Acme LLC (Identifier: ACMELLC, ID: 12347)
```

### No Active Agreement

```
Warning: No active agreement found for Acme Corporation

Ticket will be created as Time & Materials.
Proceed? [Y/n]
```

### Duplicate Detection

```
Warning: Potential duplicate ticket detected

Existing ticket #54320 "Email issues" was created 2 hours ago for this company.

Create anyway? [Y/n]
View existing ticket? [v]
```

### Board Not Found

```
Error: Service board not found: "Invalid Board"

Available boards:
- Service Desk (ID: 1)
- Managed Services (ID: 2)
- Projects (ID: 3)
```

### Contact Not at Company

```
Error: Contact "jane@other.com" not found at Acme Corporation

Contacts at this company:
- John Smith (john.smith@acme.com)
- Jane Doe (jane.doe@acme.com)
- Bob Wilson (bob@acme.com)
```

### Tool Errors

| Error | Resolution |
|-------|------------|
| Invalid board ID | List available boards with `cw_list_boards` and retry |
| Company not found | Search for correct company |
| Contact not found | Create ticket without contact |
| Rate limited | Wait and retry automatically |
| Summary too long | Truncate to 100 characters |

**No fallback path exists.** If `cw_create_ticket` fails, report the failure and
stop. Do not attempt to reach ConnectWise by any other route.

## Related Commands

- `/search-tickets` - Search existing tickets
- `/update-ticket` - Update ticket details
- `/add-note` - Add note to ticket
- `/log-time` - Log time against ticket
