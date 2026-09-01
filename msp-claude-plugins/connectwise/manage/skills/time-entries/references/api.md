# ConnectWise PSA Time Entry API Reference

**Reference, not workflow.** Every operation here is reached through a
`connectwise-manage-mcp` tool where one exists; the request shapes below
describe the ConnectWise data model so a returned record is legible. They are
not routes this plugin calls.

| Operation | Tool |
|-----------|------|
| Create | `cw_create_time_entry` |
| Get | `cw_get_time_entry` |
| Search | `cw_search_time_entries` |
| Update | **none** |
| Delete | **none** |
| Time sheets, approval | **none** |

## Create Time Entry (Start/End)

```http
POST /time/entries
Content-Type: application/json

{
  "chargeToId": 54321,
  "chargeToType": "ServiceTicket",
  "member": {"id": 123},
  "timeStart": "2024-02-15T09:00:00Z",
  "timeEnd": "2024-02-15T10:30:00Z",
  "workType": {"id": 1},
  "workRole": {"id": 2},
  "billableOption": "Billable",
  "notes": "Diagnosed email delivery issue. Identified blocked sender.",
  "addToDetailDescriptionFlag": true
}
```

## Create Time Entry (Actual Hours)

```http
POST /time/entries
Content-Type: application/json

{
  "chargeToId": 54321,
  "chargeToType": "ServiceTicket",
  "member": {"id": 123},
  "timeStart": "2024-02-15T09:00:00Z",
  "actualHours": 1.5,
  "workType": {"id": 1},
  "workRole": {"id": 2},
  "billableOption": "Billable",
  "notes": "Configured DNS records and tested mail flow."
}
```

## Create Time Entry Against Charge Code

```http
POST /time/entries
Content-Type: application/json

{
  "chargeToId": 10,
  "chargeToType": "ChargeCode",
  "company": {"id": 12345},
  "member": {"id": 123},
  "timeStart": "2024-02-15T08:00:00Z",
  "actualHours": 0.5,
  "workType": {"id": 3},
  "billableOption": "DoNotBill",
  "notes": "Weekly team meeting"
}
```

## Get Time Entry

```http
GET /time/entries/{id}
```

## Update Time Entry

**No tool updates a time entry.** The record shape is documented here for
reading; a correction is made in the PSA directly, and after the owning time
sheet is submitted it needs someone with approval rights.

## Delete Time Entry

**No tool deletes a time entry**, and the inventoried surface exposes no delete
tool at all. ConnectWise itself will not delete a billed entry.

## Search Time Entries

```http
GET /time/entries?conditions=member/id=123 and timeStart>=[2024-02-01]
```

## Time Sheets

**No tool reads or submits time sheets.** A time sheet groups entries for a
member and period; once submitted, its entries can no longer be corrected
without approval rights. Relevant when reading an entry, not actionable here.

## Approval

**No tool drives the approval workflow.** Approval state is visible on a
returned entry; changing it is a PSA action.
