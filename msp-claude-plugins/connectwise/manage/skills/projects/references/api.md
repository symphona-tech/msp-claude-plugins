# ConnectWise PSA Project API Reference

**Reference, not workflow.** Every operation here is reached through a
`connectwise-manage-mcp` tool where one exists; the request shapes below
describe the ConnectWise data model so a returned record is legible. They are
not routes this plugin calls.

| Operation | Tool |
|-----------|------|
| Create project | `cw_create_project` |
| Get project | `cw_get_project` |
| Search projects | `cw_search_projects` |
| Project tickets | `cw_search_project_tickets`, `cw_get_project_ticket` |
| Update, close, delete a project | **none** |
| Phases, templates, team assignment | **none** |

## Create Project

```http
POST /project/projects
Content-Type: application/json

{
  "name": "Office 365 Migration - ACME Corp",
  "company": {"id": 12345},
  "status": {"id": 1},
  "manager": {"id": 100},
  "estimatedStart": "2024-03-01",
  "estimatedEnd": "2024-05-01",
  "estimatedHours": 200,
  "billingMethod": "ActualRates",
  "description": "Migrate from on-premises Exchange to Office 365"
}
```

## Get Project

```http
GET /project/projects/{id}
```

## Update Project

**No tool updates a project.** The patch shape is documented for reading the
fields it names; changes are made in the PSA.

## Close Project

**No tool closes a project.** Closure is a status change, and no project update
tool exists to make it.

## Search Projects

```http
GET /project/projects?conditions=company/id=12345 and status/id=1
```

## Fixed Fee Project

```http
POST /project/projects
Content-Type: application/json

{
  "name": "Website Redesign",
  "company": {"id": 12345},
  "billingMethod": "FixedFee",
  "billingAmount": 15000.00
}
```

## Not-to-Exceed Project

```http
POST /project/projects
Content-Type: application/json

{
  "name": "System Migration",
  "company": {"id": 12345},
  "billingMethod": "NotToExceed",
  "budgetAmount": 25000.00
}
```

## Create Phase

**No tool creates or reads project phases.** This is the significant gap in
project coverage: milestone tracking, per-phase hours and overdue-phase
detection all depend on phases, and none is available.

## Get Templates

**No tool lists project templates.**

## Create Project from Template

**No tool creates a project from a template.** `cw_create_project` creates a
bare project; template expansion is a PSA action.

## Get Project Tickets

Served by `cw_search_project_tickets` (filter on `project/id`) and
`cw_get_project_ticket`. Project tickets are a **separate entity** from service
tickets; `cw_search_tickets` will not return them.

## Create Project Ticket

**No tool creates a project ticket.** `cw_create_ticket` creates a *service*
ticket, which is a different entity — the surface offers
`cw_add_project_ticket_note` for project tickets but no create.

Note also that the pre-change reference showed this as `POST /service/tickets`
carrying `project` and `phase` references. Whatever that achieved in the vendor
API, it is not reachable here and should not be treated as an equivalent.

## Team Member Assignment

**No tool assigns team members.** The assignment shape is documented for
reading a project's team; changes are made in the PSA.
