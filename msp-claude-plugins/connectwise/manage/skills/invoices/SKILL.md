---
name: "ConnectWise Manage Invoices"
description: >
  ConnectWise PSA invoices: the billing documents produced from agreements,
  time entries and products, how to search and read them, what an invoice
  record carries, and the boundary between what was billed and what was
  contracted.
when_to_use: >-
  When reading or reasoning about what a client was actually billed. Use when: connectwise
  invoice, invoice lookup connectwise, what was billed, billing history connectwise, unpaid
  invoice, invoice total, invoice date, billed hours, or reconcile invoice.
---

# ConnectWise PSA Invoices

## Overview

An invoice in ConnectWise PSA is the billing document issued to a client: the realized, financial record of what was charged for a period. It sits downstream of everything else in the PSA — agreements set what recurs, time entries and products accumulate charges, and the invoice is what the client actually receives. This skill covers searching and reading invoices, and the distinction between billed and contracted that makes invoice questions different from agreement questions.

## Anti-triggers

- **What the client is contracted for** — recurring amounts and coverage
  live on the agreement, and an agreement is not an invoice; use
  `connectwise-psa-agreements`.
- **Hours logged but not yet billed** — unbilled time is a time entry that
  has not reached the Billed state; use `connectwise-psa-time-entries`.
- **A quote or proposal** — a priced document offering future work is a CPQ
  object; use `connectwise-cpq-quotes`.
- **Expected revenue on a deal** — a forecast item is a projection, not a
  charge; use `connectwise-psa-sales`.
- **Accounting-system records** — an invoice exported to QuickBooks, Xero or
  a general ledger is a different record in a different system; this skill
  speaks only the PSA's `/finance/invoices` surface.

## Tool surface

Invoices are reached through `connectwise-manage-mcp`, never by direct REST.
The endpoint below is named for orientation only.

```
Base: /finance/invoices
```

| Tool | Purpose |
|------|---------|
| `cw_search_invoices` | Search with CW conditions syntax |
| `cw_get_invoice` | Fetch one invoice by ID |

Both are read-tier. **There is no tool that creates, updates, voids, or issues an invoice**, and none that reads an invoice's line items — see [Not available through the tool surface](#not-available-through-the-tool-surface).

## Searching

```
cw_search_invoices
  conditions: "company/id=12345 and date>=[2026-01-01]"
  orderBy:    "id desc"
  pageSize:   25
```

```
cw_get_invoice  id: 88104
```

Useful condition fields:

| Field | Matches |
|-------|---------|
| `company/id` | Exact company, once resolved with `cw_search_companies` |
| `date` | Invoice date; bracket syntax for date literals |
| `dueDate` | When payment is due |
| `status/name` | Invoice status, **by name** and per-tenant |
| `type` | Invoice type, such as an agreement or a standard invoice |
| `total`, `balance` | Amount charged and amount outstanding |

`orderBy: "id desc"` is the practical way to reach the most recent invoices, because invoice ids increase with issue order in a way `date` does not guarantee when invoices are backdated.

## Reading an invoice

An invoice record typically carries its number, type, status, company, bill-to details, invoice and due dates, the period it covers, the agreement it was produced from where there is one, and the monetary fields — subtotal, tax, total, payments applied, and remaining balance.

**`balance` is the field an "unpaid invoices" question turns on**, not `status`. Status vocabularies are per-tenant and an instance may or may not move an invoice to a paid status automatically; a non-zero balance is the fact.

## Billed is not contracted, and not logged

Three records describe money in this PSA and they answer different questions:

| Question | Record | Reachable here |
|----------|--------|----------------|
| What is the client contracted for? | Agreement | [ConnectWise Agreements](../agreements/SKILL.md) |
| What work was performed? | Time entry | [ConnectWise Time Entries](../time-entries/SKILL.md) |
| What was the client charged? | Invoice | This skill |

They do not reconcile to each other on this surface. An invoice names the agreement it came from, and **nothing joins an invoice to the time entries inside it** — so *"which hours are on this invoice"* cannot be answered here, and an answer assembled by matching dates is a guess wearing the shape of a reconciliation.

## Gotchas

- **An invoice total is not a client's outstanding balance.** One invoice's `balance` is that invoice's; a client-level figure is the sum across a full, paged search, which means a partial page silently under-reports.
- **Paging matters more here than elsewhere.** A year of monthly invoicing across a client base exceeds the default `pageSize` of 25 immediately; set it deliberately and page to exhaustion before summing anything.
- **Invoice status is per-tenant configuration with no lookup.** No tool enumerates invoice statuses, so a status filter is an unvalidatable name match with the same ambiguity configurations have — an empty result may mean the filter names nothing.
- **Credits and adjustments appear as invoice records too.** A negative total is a credit memo in most configurations rather than an error; do not filter it out of a billing history without saying so.
- **The line items are not readable.** An invoice answers *how much*; it does not answer *for what* on this surface.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Invoice line items | An invoice line-item tool. **None exists**, so an invoice is a header here |
| The time entries billed on an invoice | A join between invoices and time entries. Neither side is queryable by the other |
| Enumerating invoice statuses | An invoice-status tool |
| Creating, updating, voiding, or issuing an invoice | A write tool. **The surface exposes none** — both invoice tools here are read-tier |
| Recording or reading a payment | A payment tool |

**The line-item gap sets the ceiling on what an invoice answer can be.** Totals, dates, status, balance and the owning agreement are all readable; the composition of the charge is not. A question of the form *"why is this invoice this size"* cannot be answered on this surface, and should be handed to someone with the PSA in front of them rather than approximated from time entries.

**Invoicing is read-only here by the whole surface, not by configuration.** No amount of permission grants a write, because there is no write tool to grant.

## Related Skills

- [ConnectWise Agreements](../agreements/SKILL.md) — what recurs, as distinct from what was billed
- [ConnectWise Time Entries](../time-entries/SKILL.md) — the work behind a charge
- [ConnectWise Companies](../companies/SKILL.md) — resolving the billed company
- [ConnectWise Sales](../sales/SKILL.md) — forecast revenue, as distinct from realized
- [ConnectWise API Patterns](../api-patterns/SKILL.md) — conditions syntax and paging
