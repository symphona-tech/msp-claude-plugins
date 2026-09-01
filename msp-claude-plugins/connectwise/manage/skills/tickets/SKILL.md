---
name: "ConnectWise Manage Tickets"
description: >
  ConnectWise PSA ticket management: ticket fields, service boards, statuses,
  priorities, SLAs, ticket notes, and workflow automation. Essential for MSP
  technicians handling service delivery through ConnectWise PSA.
when_to_use: >-
  When creating, updating, searching, or managing service desk operations. Use when: connectwise
  ticket, connectwise psa ticket, service ticket connectwise, create ticket connectwise, ticket
  board, ticket status connectwise, ticket priority, connectwise service desk, ticket triage,
  escalate ticket, resolve ticket, ticket notes, sla calculation, or ticket workflow.
---

# ConnectWise PSA Ticket Management

Tickets are the core unit of service delivery in ConnectWise PSA. Every client request, incident, and change becomes a service ticket, created through `cw_create_ticket`. This skill covers creating, searching, updating, and closing tickets with proper SLA handling.

> For complete field definitions see [FIELDS.md](./FIELDS.md). For status values, priorities, SLA tables, and note types see [REFERENCE.md](./REFERENCE.md).

## Anti-triggers

- **RMM-generated alerts** — "connectwise alert" commonly means an
  Automate monitor alert, which lives in the RMM and only becomes a PSA
  ticket once escalated; use `connectwise-automate-alerts`.
- **Project tickets** — tickets on a project board are `/project/tickets`,
  a separate entity with its own tools (`cw_search_project_tickets`); use
  `connectwise-psa-projects`.
- **Hours worked on a ticket** — the ticket is the charge target, not the
  record that holds the time; use `connectwise-psa-time-entries`.
- **Another PSA's tickets** — Autotask, HaloPSA, Atera, Syncro and
  SuperOps all call them tickets; this skill speaks only the ConnectWise
  PSA `/service/tickets` API. Use `autotask-tickets`, `halopsa-tickets`,
  `atera-tickets`, `syncro-tickets` or `superops-tickets`.

## Core API Operations

Every operation below runs through a `connectwise-manage-mcp` tool. The
endpoint each one corresponds to is named for orientation only — it is not a
route this plugin calls.

### Create a Ticket

```
cw_create_ticket
  summary:            "Unable to access email - multiple users affected"
  boardId:            1
  companyId:          12345
  contactId:          67890
  priorityId:         2
  initialDescription: "Sales team (5 users) reporting Outlook disconnected since 9am."
```

Only `summary` is required, and the tool takes resolved numeric IDs rather than
nested objects. Use `initialDescription` for the first note with full details.

### Get / Update a Ticket

```
cw_get_ticket  id: 54321
```

```
cw_update_ticket
  id: 54321
  operations: [
    {"op": "replace", "path": "status/name", "value": "In Progress"},
    {"op": "replace", "path": "owner/id",    "value": 123}
  ]
```

### Add a Note

```
cw_add_ticket_note
  id:                   54321
  text:                 "Identified the issue as a DNS configuration problem."
  internalAnalysisFlag: true
```

Set `resolutionFlag: true` for resolution notes, and `customerUpdatedFlag: true`
for notes the customer should see on the portal.

### Search Tickets

```
cw_search_tickets
  conditions: "company/id=12345 and status/name!=\"Closed\""
  orderBy:    "priority/id asc"
```

### Read Notes

```
cw_get_ticket_notes  id: 54321
```

## Common Query Patterns

```
# Open tickets for a company
conditions=company/id=12345 and closedFlag=false

# High priority open tickets
conditions=priority/id<=2 and closedFlag=false

# Tickets by date range
conditions=dateEntered>=[2024-01-01] and dateEntered<[2024-02-01]

# SLA-breached tickets
conditions=_info/sla_resolve_by<[2024-02-15T12:00:00Z] and closedFlag=false

# My assigned tickets
conditions=resources contains "jsmith" and closedFlag=false
```

## Workflow: Ticket Closure with Validation

**There is no close tool.** Closure is composed, and each checkpoint is a
separate call:

1. **Verify resolution exists** — `cw_get_ticket` and confirm the `resolution`
   field is populated. If empty, add a resolution note first with
   `cw_add_ticket_note` and `resolutionFlag: true`.
2. **Check time entries** — `cw_search_time_entries` with
   `conditions=chargeToId=54321 and chargeToType="ServiceTicket"`. Ensure all
   work is logged.
3. **Confirm status is Completed** — tickets must pass through `Completed`
   before `Closed`. Resolve valid statuses with `cw_list_statuses` for the
   ticket's board.
4. **Close the ticket** —

   ```
   cw_update_ticket
     id: 54321
     operations: [
       {"op": "replace", "path": "status/id",  "value": <closed_status_id>},
       {"op": "replace", "path": "resolution", "value": "DNS records corrected."}
     ]
   ```

5. **Validate closure** — `cw_get_ticket` and confirm `closedFlag: true` and
   `closedDate` is populated.

> If any checkpoint fails, stop and resolve the issue before proceeding.
> Closing without a resolution note leaves the ticket incomplete in reports.
> These are five separate calls rather than one transaction, so a failure part
> way through leaves the ticket in whatever state the last successful call put
> it — report which steps completed.

## Best Practices

1. **Always specify board** — determines available statuses, workflows, and SLA rules.
2. **Set accurate priority** — use impact/urgency; Priority 1 = highest (business down). See [REFERENCE.md](./REFERENCE.md) for the matrix.
3. **Search before creating** — check for duplicates with `conditions=company/id={id} and summary contains "keyword"`.
4. **Update status promptly** — keeps SLA clocks accurate and queues reliable.

## Related Skills

- [ConnectWise Companies](../companies/SKILL.md) — Company management
- [ConnectWise Contacts](../contacts/SKILL.md) — Contact management
- [ConnectWise Time Entries](../time-entries/SKILL.md) — Time tracking
- [ConnectWise API Patterns](../api-patterns/SKILL.md) — Query syntax and auth
