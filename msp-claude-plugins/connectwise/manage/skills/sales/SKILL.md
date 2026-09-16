---
name: "ConnectWise Manage Sales"
description: >
  ConnectWise PSA sales pipeline: opportunities, pipeline stages, forecast and
  revenue items, opportunity notes, and sales activities — the follow-up
  records that track outreach against a company, contact or deal.
when_to_use: >-
  When reading or reasoning about the sales pipeline and its follow-up records. Use when:
  connectwise opportunity, sales pipeline connectwise, pipeline stage, sales stage, deal
  connectwise, forecast connectwise, revenue item, opportunity note, sales activity, log an
  activity connectwise, follow-up record, or won lost deal.
---

# ConnectWise PSA Sales

## Overview

The sales side of ConnectWise PSA tracks work before it is contracted: an **opportunity** is a deal being pursued, a **stage** is where it sits in the pipeline, a **forecast item** is the revenue expected from it, and an **activity** is a piece of follow-up recorded against a company, contact or deal. This skill covers reading the pipeline and creating activities, which is the one write this surface exposes here.

## Anti-triggers

- **A signed contract** — once a deal is won the record that governs
  delivery and billing is an agreement, not an opportunity; use
  `connectwise-psa-agreements`.
- **A quote or proposal document** — pricing documents are CPQ objects with
  their own tools; use `connectwise-cpq-quotes`.
- **A calendar or dispatch entry** — an activity is a sales follow-up
  record, not a scheduled appointment. Despite what its tool description
  says, **`cw_search_activities` reads `/sales/activities`**; a schedule
  entry is a different entity with no tool at all. See
  [Activities are sales records](#activities-are-sales-records).
- **Service work** — a client asking for help is a ticket; use
  `connectwise-psa-tickets`.
- **Money already billed** — realized revenue is an invoice, not a forecast;
  use `connectwise-psa-invoices`.

## Tool surface

Sales records are reached through `connectwise-manage-mcp`, never by direct
REST. The endpoints below are named for orientation only.

```
Base: /sales/opportunities, /sales/stages, /sales/activities
```

| Tool | Purpose | Tier |
|------|---------|------|
| `cw_search_opportunities` | Search opportunities with CW conditions syntax | read |
| `cw_get_opportunity` | Fetch one opportunity by ID | read |
| `cw_search_opportunity_forecasts` | Revenue/forecast items for one opportunity | read |
| `cw_search_opportunity_notes` | Notes on one opportunity | read |
| `cw_search_sales_stages` | List the pipeline stages | read |
| `cw_search_activities` | Search sales activities | read |
| `cw_get_activity` | Fetch one activity by ID | read |
| `cw_create_activity` | Create a sales activity | **write** |

`cw_create_activity` is the only write tool in this skill. Under this distribution's effective policy a write-tier tool is gated by a confirmation prompt before it runs; treat creating an activity as a change to PSA state that somebody will see.

## Reading the pipeline

```
cw_search_opportunities
  conditions: "company/id=12345 and closedFlag=false"
  orderBy:    "expectedCloseDate asc"
```

```
cw_get_opportunity                id: 3310
cw_search_opportunity_forecasts   opportunityId: 3310
cw_search_opportunity_notes       opportunityId: 3310
cw_search_sales_stages
```

**The forecast and note tools take `opportunityId` and nothing else** beyond paging — they are sub-resource reads, not searches, so they cannot be filtered and cannot be run across the pipeline in one call. Answering *"what is the forecast for this quarter"* means selecting opportunities first and then reading each one's forecast items.

`cw_search_sales_stages` is the one enumeration this skill has, and it is worth using rather than assuming: stage names are per-tenant, and `status/name = '1. Open'` in the vendor's own example is a naming convention rather than a guarantee.

## Opportunity fields worth knowing

| Field | Why it matters |
|-------|----------------|
| `closedFlag` | The pipeline filter. A won and a lost deal are both closed |
| `status/name` | Distinguishes won from lost, and is per-tenant |
| `stage/name` | Where in the pipeline; resolve the vocabulary with `cw_search_sales_stages` |
| `expectedCloseDate` | The date every forecast question turns on |
| `probability` | Weighting applied to forecast revenue |
| `company`, `contact` | Who the deal is with |

**`closedFlag=false` is the open-pipeline filter; it is not "active" in the agreement sense** — there is no expiry to check here, which is the opposite of the trap agreements set.

## Activities are sales records

`cw_search_activities` describes itself as searching *schedule* activities. **It reads `/sales/activities`**, and so do `cw_get_activity` and `cw_create_activity`. An activity is a follow-up record against a company, contact or opportunity — a call made, an email sent, a meeting held — and appears on a member's schedule as a side effect rather than as its purpose.

**This matters because a semantically adjacent tool is not a substitute.** A request to schedule an appointment or dispatch a technician is a schedule entry, `/schedule/entries`, and **no tool creates one**. Creating an activity instead would put a sales follow-up where an appointment belongs, and it would be the wrong record in the wrong module.

```
cw_search_activities
  conditions: "company/id=12345 and dateStart>=[2026-09-01]"
  orderBy:    "dateStart desc"
```

```
cw_get_activity  id: 8842
```

### Creating an activity

```
cw_create_activity
  name:      "Renewal discussion - Acme MSA"
  companyId: 12345
  contactId: 6677
  memberId:  217
  dateStart: "2026-09-18T14:00:00Z"
  dateEnd:   "2026-09-18T14:30:00Z"
  notes:     "Discussed block-hour top-up ahead of the December renewal"
```

Only `name` is required. `memberId` is **not inferred** — the server authenticates as one API member, which is not the person the activity belongs to, so resolve it with `cw_search_members` and never default to the service account. `typeId` selects the activity type and **cannot be resolved by name**, because no tool enumerates activity types; leave it unset and let the instance default apply rather than guessing an id.

## Gotchas

- **The tool description for `cw_search_activities` says "schedule activities" and the endpoint is `/sales/activities`.** Read the endpoint, not the description. This is upstream's wording and is corrected here rather than in the server.
- **Forecast items are per opportunity.** There is no pipeline-wide forecast read, so any aggregate is assembled client-side from individual calls and is only as complete as the opportunity list it started from.
- **Stage and status are different fields.** A deal can be at a late stage and still open; `closedFlag` is what says it finished, and `status/name` is what says how.
- **An opportunity cannot be created, updated, or closed here.** Reading the pipeline is possible; moving a deal through it is not.
- **Activity types are per-tenant configuration with no lookup**, which is the same shape as the configuration type and status gaps — a hardcoded id is drift rather than a shortcut.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Creating, updating, or closing an opportunity | An opportunity write tool. **The surface exposes none** |
| Adding a note to an opportunity | An opportunity-note write tool; `cw_search_opportunity_notes` reads only |
| Updating or deleting an activity | An activity update or delete tool. **Neither exists** |
| Resolving an activity type by name | An activity-type tool |
| Pipeline-wide forecast in one call | A forecast search that is not scoped to one `opportunityId` |
| Creating a schedule entry | `POST /schedule/entries`. **No tool serves it**, and `cw_create_activity` is not a substitute |
| Enumerating schedule types or statuses | A schedule-type and schedule-status tool |

**The pipeline is readable and not movable.** Everything about a deal can be reported; nothing about it can be advanced, which makes this surface an analysis tool rather than a CRM client.

**The schedule gap is the one most likely to be papered over.** The names invite the substitution — an activity has a start date, an end date and an assigned member, and looks like an appointment — and it is the wrong record. A request to schedule work has no home on this surface; say so rather than creating an activity that resembles one.

## Related Skills

- [ConnectWise Companies](../companies/SKILL.md) — the account a deal belongs to
- [ConnectWise Contacts](../contacts/SKILL.md) — the person a deal is with
- [ConnectWise Agreements](../agreements/SKILL.md) — what a won deal becomes
- [ConnectWise Invoices](../invoices/SKILL.md) — revenue actually billed
- [ConnectWise API Patterns](../api-patterns/SKILL.md) — conditions syntax and paging
