---
name: "ConnectWise Manage Agreements"
description: >
  ConnectWise PSA finance agreements: recurring revenue contracts, agreement
  types, additions (line items), prepaid hours and block time, expiry and
  renewal, and what "covered" does and does not mean when answering a coverage
  question.
when_to_use: >-
  When reading, searching, or reasoning about contracts and entitlements. Use when: connectwise
  agreement, managed services agreement, msa connectwise, is this covered, agreement coverage,
  prepaid hours, block hours, block time, agreement addition, agreement line item, contract
  expiry connectwise, agreement renewal, or recurring revenue contract.
---

# ConnectWise PSA Agreements

## Overview

An agreement in ConnectWise PSA is the contract between the MSP and a client: what is covered, for how long, at what price, and against which prepaid balance work is drawn. Agreements drive billing on every time entry and ticket that references them, which is why a wrong answer about coverage is a billing error rather than a reporting one. This skill covers reading agreements, their line items, and the specific shape of the coverage question this tool surface can and cannot answer.

## Anti-triggers

- **A quote or a proposal** — a document offering work that is not yet
  contracted is a CPQ quote, not an agreement; use `connectwise-cpq-quotes`.
- **An invoice** — the agreement says what recurs, the invoice says what was
  actually billed in a period; use `connectwise-psa-invoices`.
- **An opportunity** — a deal being pursued is pre-contract and lives in the
  sales pipeline; use `connectwise-psa-sales`.
- **Hours already worked** — time entries are the consumption record and are
  not reachable from an agreement here; use `connectwise-psa-time-entries`.
- **Covered devices** — what a client owns is a configuration item, and
  configuration coverage is a separate relationship; use
  `connectwise-psa-configurations`.

## Tool surface

Agreements are reached through `connectwise-manage-mcp`, never by direct REST.
The endpoint below is named for orientation only.

```
Base: /finance/agreements
```

| Tool | Purpose |
|------|---------|
| `cw_search_agreements` | Search with CW conditions syntax |
| `cw_get_agreement` | Fetch one agreement by ID |
| `cw_get_agreement_additions` | List the line items on one agreement |

All three are read-tier. **There is no tool that creates, updates, or cancels an agreement**, and none that reports usage against one — see [Not available through the tool surface](#not-available-through-the-tool-surface).

## Finding an active agreement

**`cancelledFlag = false` alone is not "active".** An agreement that was never cancelled but whose `endDate` has passed is expired, and reporting it as active overstates coverage. An open-ended agreement carries a null `endDate`, so both branches are needed:

```
cw_search_agreements
  conditions: "company/id=12345 and cancelledFlag=false and (endDate >= [2026-09-16] or endDate = null)"
```

To include expired agreements deliberately, drop only the date clauses and keep `cancelledFlag=false` — then label every result with its `endDate` so the reader can see which are current.

```
cw_get_agreement             id: 9876
cw_get_agreement_additions   agreementId: 9876
```

`cw_get_agreement_additions` pages independently of the agreement, so a long addition list needs `page` and `pageSize` rather than arriving whole.

## Agreement types

| Type | What it means | Typical billing |
|------|---------------|-----------------|
| Managed Services | Ongoing coverage for a defined scope | Recurring, monthly |
| Block Hours | A prepaid pool of hours drawn down by time entries | Prepaid, topped up by an addition |
| Block Money | A prepaid monetary balance | Prepaid |
| Time & Materials | No prepayment; work billed as performed | Per hour, at the work role's rate |
| Project | Scoped to one project's budget | Fixed or milestone |

**The table is illustrative, not an enumeration of your instance.** No tool lists agreement types, so type names come back on the record rather than from a lookup — read `type/name` off the agreement instead of assuming this vocabulary.

## Additions are what the agreement actually covers

An addition is a line item: a recurring service, a one-time charge, or a block of hours added to the pool. An agreement's headline name says what it is called; its additions say what it contains.

A coverage answer that names the agreement and omits its additions is describing the wrapper rather than the contents. Call `cw_get_agreement_additions` whenever the question is *"what does this client get"* rather than *"which contract are they on"*.

Additions carry a `unitOfMeasure` describing how the line bills. **No tool resolves a unit of measure by name**, so the value is readable on the addition and not selectable anywhere.

## Answering "is this work covered?"

This is the question agreements exist for, and on this tool surface it can be answered **partially and honestly** or completely and wrongly. The parts that are reachable:

1. **Is there a current agreement?** `cw_search_agreements` with both clauses above.
2. **What does it contain?** `cw_get_agreement_additions`.
3. **How much block time was purchased?** Readable from the agreement and its block-hour additions.

The parts that are not reachable are in the table at the end of this skill, and the two that matter most are **usage** and **work-role coverage**. Report the answer you have and name the parts you do not — a coverage summary that silently omits role coverage invites the reader to conclude that covered hours are the whole answer.

## Gotchas

- **An agreement's prepaid balance is not the same as its remaining hours.** Purchased hours come from block additions; consumed hours come from time entries, which cannot be scoped to an agreement here. A "remaining" figure computed from purchased hours alone is purchased hours wearing a different label.
- **`company/id` filters, `company/name` matches loosely.** Prefer the id once it is resolved with `cw_search_companies`; a name condition matches every company sharing a prefix.
- **A child agreement inherits from its parent.** `parentAgreement` on the record means coverage may be defined a level up, so reading the child alone can under-report it.
- **Cancelled is not deleted.** A cancelled agreement stays readable and still carries its history, which is why every active-agreement query needs the flag.
- **Dates are instance-local.** `[2026-09-16]` bracket syntax in a condition is evaluated by the server; see [ConnectWise API Patterns](../api-patterns/SKILL.md) for the conditions grammar.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Usage against an agreement | Time entries scoped to an agreement. `cw_search_time_entries` charges to a **ticket**, and nothing resolves an agreement's tickets |
| Which work roles an agreement covers | An agreement work-role tool |
| Which work types an agreement covers | An agreement work-type tool |
| Resolving a unit of measure by name | A unit-of-measure tool |
| Creating, updating, or cancelling an agreement | A write tool. **The surface exposes none** — every agreement tool here is read-tier |

**Usage is the largest single gap in the agreement story.** A coverage answer without it is about entitlement alone: it can say what was bought and cannot say what is left. Computing usage needs a separate `cw_search_time_entries` call filtered to the company's tickets, which is a different question with a different answer — do not present an agreement response as though it carried usage.

**Work-role and work-type coverage is the gap that changes answers.** *"Is this work covered"* frequently turns on the role performing it, so a coverage answer built here is about hours and additions only. Say so whenever billing coverage is the question being asked.

**None of these is closed by reading the ConnectWise REST documentation.** The endpoints exist in the vendor API; they are not in this server's tool surface, and this plugin calls no route directly.

## Related Skills

- [ConnectWise Companies](../companies/SKILL.md) — resolving the company an agreement belongs to
- [ConnectWise Time Entries](../time-entries/SKILL.md) — the consumption side, and why it cannot be joined here
- [ConnectWise Invoices](../invoices/SKILL.md) — what was actually billed
- [ConnectWise Configurations](../configurations/SKILL.md) — the assets a client owns
- [ConnectWise API Patterns](../api-patterns/SKILL.md) — conditions syntax and paging
