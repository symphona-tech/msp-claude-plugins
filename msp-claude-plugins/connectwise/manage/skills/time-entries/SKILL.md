---
name: "ConnectWise Manage Time Entries"
description: >
  ConnectWise PSA time entry management: charge-to types (tickets, projects,
  charge codes), billable vs non-billable time, work types and work roles,
  time sheets, and the time entry approval workflow.
when_to_use: >-
  When creating, updating, searching, or managing time tracking. Use when: connectwise time entry,
  time tracking connectwise, log time connectwise, billable time, non-billable time, work type,
  work role, time sheet, time approval, or hours logged.
---

# ConnectWise PSA Time Entry Management

## Overview

Time entries in ConnectWise PSA track time spent on tickets, projects, and other activities. Accurate time tracking is essential for billing, resource management, and profitability analysis. This skill covers time entry CRUD operations, work types, work roles, billing settings, and approval workflows.

## Anti-triggers

- **Writing up what happened on a ticket** — narrative goes in a ticket
  note (`cw_add_ticket_note`); only billable duration belongs in a time
  entry. Use `connectwise-psa-tickets`.
- **Project time** — project hours charge to a `ProjectTicket`, so the
  ticket has to be located first; use `connectwise-psa-projects`.

## Tool surface

Time entries are reached through `connectwise-manage-mcp`, never by direct
REST. The endpoint below is named for orientation only.

```
Base: /time/entries
```

| Tool | Purpose |
|------|---------|
| `cw_search_time_entries` | Search with CW conditions syntax |
| `cw_get_time_entry` | Fetch one entry by ID |
| `cw_create_time_entry` | Create an entry |

**There is no update tool and no delete tool for time entries**, and no way to
set the billing option — see
[Not available through the tool surface](#not-available-through-the-tool-surface).

## Time Entry Types

Time can be logged against different record types:

| Charge To Type | Description |
|----------------|-------------|
| `ServiceTicket` | Time against service tickets |
| `ProjectTicket` | Time against project tickets |
| `ChargeCode` | Time against charge codes (internal) |
| `Activity` | Time against activities |

See [references/fields.md](references/fields.md) for the complete time entry field reference.

## Work Types

Work types categorize the nature of work performed.

| Type | Description | Typical Billing |
|------|-------------|-----------------|
| Regular | Normal work hours | Billable |
| Overtime | After-hours work | 1.5x rate |
| Training | Training time | Non-billable |
| Travel | Travel time | Varies |
| Remote | Remote support | Billable |
| On-site | On-site work | Billable |
| Administrative | Admin tasks | Non-billable |

**No tool enumerates work types**, and none resolves one by name, so `workTypeId` cannot be looked up here. The table above is illustrative; leave the field unset and let the agreement and board defaults apply.

## Work Roles

Work roles determine billing rates based on skill level.

| Role | Description | Typical Rate |
|------|-------------|--------------|
| Level 1 Tech | Help desk | $75-100/hr |
| Level 2 Tech | Desktop support | $100-125/hr |
| Level 3 Tech | Systems admin | $125-150/hr |
| Engineer | Senior engineer | $150-200/hr |
| Consultant | Expert consultant | $200-250/hr |
| Project Manager | PM work | $125-175/hr |

**No tool enumerates work roles**, and none resolves one by name. This matters more than the work-type gap: the work role sets the billing rate, so a guessed id bills the customer wrongly and nothing downstream flags it. Leave `workRoleId` unset.

## Billing Options

| Option | Description |
|--------|-------------|
| `Billable` | Time is billable at standard rate |
| `DoNotBill` | Time excluded from billing |
| `NoCharge` | Time shows on invoice at $0 |
| `NoDefault` | Use ticket/agreement default |

### How Billing Is Determined

Precedence order, first match wins:

1. Time entry `billableOption` (if set)
2. Ticket `billTime` setting
3. Agreement billing rules
4. Company default

## Common Workflows

### Find time on a ticket

```
cw_search_time_entries
  conditions: "chargeToId=54321 and chargeToType=\"ServiceTicket\""
```

`chargeToType` is one of `ServiceTicket`, `ProjectTicket`, `ChargeCode` or
`Activity`. This is a search across all entries filtered to the target, not a
ticket sub-resource, so it pages independently.

### Log time against a ticket

```
cw_create_time_entry
  chargeToType: "ServiceTicket"
  chargeToId:   54321
  memberId:     217
  timeStart:    "2026-02-04T09:00:00Z"
  timeEnd:      "2026-02-04T11:30:00Z"
  notes:        "Remote support session"
```

`chargeToType`, `chargeToId`, `memberId` and `timeStart` are all **required**.
`memberId` is not inferred — the server authenticates as one API member, which
is not the technician the time belongs to, so resolve it with
`cw_search_members` and never default to the service account.

**A time entry against a ticket lands on the customer's next invoice.** Treat
creating one as a billing action.

## Reading time back

```
cw_search_time_entries  conditions: "member/identifier=\"jtech\" and timeStart>=[2026-02-01]"
cw_get_time_entry       id: 11223
```

A returned entry carries `billableOption`, `actualHours`, `hoursBilled`,
`workType`, `workRole` and the agreement fields — so billing treatment is
**readable** even though it cannot be set.

## Charge Codes

Charge codes are used for non-ticket time (meetings, training, etc.). **No tool
enumerates them**, so the table below is illustrative rather than a validated
list for your instance; confirm the codes configured in the PSA.

| Code | Description | Billable |
|------|-------------|----------|
| MTNG | Internal meetings | No |
| TRNG | Training | No |
| ADMIN | Administrative | No |
| PTO | Paid time off | No |
| PROJ | Project work | Yes |
| ONCALL | On-call time | Varies |

## Gotchas

- **Billed time entries cannot be deleted.** The API rejects a delete once `status` is `Billed`, and the invoice has to be adjusted instead. No delete tool is exposed here in any case, billed or not.
- **`company` is required for `ChargeCode` entries only.** `ServiceTicket`/`ProjectTicket` entries infer the company from the ticket.
- **`billableOption` on the time entry wins over ticket/agreement defaults** — see billing precedence above. A blank or `NoDefault` value falls through to the ticket, then the agreement, then the company.
- Time sheets and time entries use different status vocabularies. Time sheets go `Open -> Submitted -> Approved/Rejected`; time entries go `Open -> Approved/Rejected -> Billed` (no `Submitted` state). Approving a time sheet does not retroactively change the `status` of every entry inside it. See [references/api.md](references/api.md) for both status tables.

## Best Practices

1. **Be specific in notes** - Document what was done for invoice clarity
2. **Use correct work type** - Important for accurate billing rates
3. **Set appropriate work role** - Affects billing rate
4. **Mark non-billable correctly** - Don't inflate billable hours
5. **Use charge codes** - For internal time tracking

See [references/errors.md](references/errors.md) for the complete error reference.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Setting the billing option on a new entry | A billing argument on `cw_create_time_entry` |
| Updating a time entry | An update tool. **None exists** |
| Deleting a time entry | A delete tool. **None exists** |
| Resolving a work type or work role by name | A work type or work role tool |
| Approval workflow | An approval tool |
| Looking up charge codes | A charge-code tool |

**The billing gap is the one that costs money.** `cw_create_time_entry` takes no
billing argument, so an entry created here receives the server's default
treatment — billable, in most configurations. The Billing Options section above
documents what the values mean when **reading** an entry; none of them can be
set. If work must not be billed, do not create the entry here.

**And it cannot be corrected afterwards.** With no update and no delete tool, a
wrong entry is a manual fix in the PSA — and once the owning time sheet is
submitted, a fix by someone with approval rights. Get it right the first time.

The work type and work role gaps compound this: the work role sets the billing
**rate**, and it cannot be resolved by name either.

## Related Skills