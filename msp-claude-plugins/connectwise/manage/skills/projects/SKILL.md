---
name: "ConnectWise Manage Projects"
description: >
  ConnectWise PSA project management: project lifecycle and status/type
  values, phases, templates, resource/team allocation, budgeting, billing
  methods, and project tickets.
when_to_use: >-
  When creating, updating, managing project phases, templates, and resource allocation. Use when:
  connectwise project, project management, create project connectwise, project phase, project
  template, project resource, project budget, project billing, project ticket, or project
  schedule.
---

# ConnectWise PSA Project Management

## Overview

Projects in ConnectWise PSA track larger bodies of work that span multiple tickets, phases, and resources. Projects support templates, phases, budgeting, resource allocation, and various billing methods. This skill covers project CRUD operations, phases, templates, resources, and project tickets.

## Anti-triggers

- **Service desk work** — break/fix tickets on a service board live at
  `/service/tickets` and are a different entity from project tickets; use
  `connectwise-psa-tickets`.
- **Hours booked against a project** — time charges to a `ProjectTicket`,
  never to the project or phase directly; use
  `connectwise-psa-time-entries`.

## Tool surface

Projects are reached through `connectwise-manage-mcp`, never by direct REST.
The endpoints below are named for orientation only.

```
Projects:        /project/projects
Project tickets: /project/tickets
```

| Tool | Purpose |
|------|---------|
| `cw_search_projects` | Search projects with CW conditions syntax |
| `cw_get_project` | Fetch one project by ID |
| `cw_create_project` | Create a project |
| `cw_search_project_tickets` | Search tickets on a project board |
| `cw_get_project_ticket` | Fetch one project ticket |
| `cw_get_project_ticket_notes` | Read a project ticket's notes |
| `cw_add_project_ticket_note` | Add a note to a project ticket |

There is **no tool for project phases, templates, or team assignment** — see
[Not available through the tool surface](#not-available-through-the-tool-surface).

## Key Concepts

### Project Status Values

Standard project statuses in ConnectWise PSA:

| Status ID | Name | Description |
|---------|------|-------------|
| 1 | Open | Active project |
| 2 | Closed | Completed project |
| 3 | On Hold | Temporarily paused |
| 4 | Cancelled | Cancelled project |
| 5 | Waiting | Awaiting approval/resources |

Project statuses are configurable. No tool enumerates them, so treat the table above as illustrative and confirm the values configured in your own PSA.

### Project Types

| Type ID | Name | Description |
|---------|------|-------------|
| 1 | Project | Standard project |
| 2 | Template | Project template |

See [references/fields.md](references/fields.md) for the complete project, phase, project ticket, and team member field reference.

### Billing Methods

| Method | Description |
|--------|-------------|
| `ActualRates` | Bill at standard work role rates |
| `FixedFee` | Fixed project price |
| `NotToExceed` | Actual rates with cap |
| `OverrideRate` | Custom hourly rate |

## Common Workflows

### Find projects

```
cw_search_projects
  conditions: "status/id=1"
  orderBy:    "estimatedEnd asc"
```

Status 1 is Open. See the status table above for the rest.

### Read one project

```
cw_get_project  id: 4321
```

The record carries `estimatedHours`, `actualHours`, `percentComplete` and
`budgetAnalysis`, which is what most delivery-health questions turn on.

### Create a project

```
cw_create_project
  name:      "Acme Server Migration"
  companyId: 12345
```

Check the tool's input schema for the fields it accepts; the concepts above
describe the full ConnectWise model, which is wider.

### Work a project ticket

```
cw_search_project_tickets  conditions: "project/id=4321 and closedFlag=false"
cw_get_project_ticket      id: 98765
cw_add_project_ticket_note id: 98765  text: "Cutover scheduled for Friday."
```

Project tickets are a **separate entity** from service tickets and have their
own tools; `cw_search_tickets` will not find them.

## Reading project health

Budget signals come off the project record itself, so a portfolio review is one
`cw_search_projects` call plus arithmetic:

- `budgetAnalysis` — `OverBudget`, `OnBudget` or `UnderBudget`
- burn rate — `(actualHours / estimatedHours) / (percentComplete / 100)`; above
  1.2 means hours are being consumed faster than progress is delivered
- remaining budget — `estimatedHours - actualHours`, which matters most on
  `FixedFee` and `NotToExceed` work

**Phase-level health is not observable here.** A project can read healthy at the
top level while individual phases run overdue, and no tool exposes phases.

## Gotchas

- **Templates are just projects with a template project type.** There is no separate template tool; `cw_search_projects` finds them by filtering on the type, e.g. `conditions=type/name="Template"`. Resolve the type for your own instance rather than assuming an id — project types are configurable.
- **`projectTemplateId` only applies at creation.** It copies phases, tickets, budget/billing settings, and team assignments once; it does not keep the new project in sync with later template edits.
- **`budgetAnalysis` is server-calculated**, not settable — use `budgetHours`/`budgetAmount` to set the cap and read `budgetAnalysis` to see whether the project is over/under/on budget.
- **Project tickets are a separate entity from service tickets.** They are read with `cw_search_project_tickets`; no tool creates one. Do not reach for `cw_create_ticket` — that writes a service ticket, which is not the same record.

## Best Practices

1. **Use templates** - Create templates for repeatable projects
2. **Define phases** - Break large projects into phases
3. **Set realistic budgets** - Include contingency time
4. **Assign project manager** - Every project needs an owner
5. **Link to agreement** - For managed services project work
6. **Track completion %** - Update regularly for visibility
7. **Use WBS codes** - Helps with reporting and organization
8. **Close completed projects** - Don't leave finished projects open

See [references/errors.md](references/errors.md) for the complete error reference.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Reading or creating project phases | A phases tool |
| Creating a project from a template | A project-template tool |
| Assigning team members | A project-team tool |
| Updating or deleting a project | An update or delete tool. `cw_create_project` exists; neither counterpart does |

**Phases are the significant gap.** Milestone tracking, per-phase hours and
overdue-phase detection all depend on them, and none is available. Where a
review would report on phases, say so rather than substituting project-level
totals — the two answer different questions, and a project-level read presented
as a delivery picture is the failure mode worth avoiding.

## Related Skills