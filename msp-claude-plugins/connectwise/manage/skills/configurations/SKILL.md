---
name: "ConnectWise Manage Configurations"
description: >
  ConnectWise PSA configuration items: the asset and CI records that describe
  what a client owns, configuration types and statuses, serial and tag
  identifiers, company and site association, and the relationship between a
  configuration and the tickets raised against it.
when_to_use: >-
  When looking up, searching, or reasoning about client assets in the PSA. Use when: connectwise
  configuration, configuration item, connectwise ci, connectwise asset, client asset, serial
  number lookup, tag number, asset inventory connectwise, what does this client own, device
  record connectwise, or configuration type.
---

# ConnectWise PSA Configuration Items

## Overview

A configuration item — a CI, or simply a configuration — is the PSA's record of something a client owns: a server, a workstation, a firewall, a licence, a line of connectivity. It is the asset side of the service relationship, and it is what a ticket points at when the question is *which machine*. This skill covers searching configurations, reading one in full, and the two lookup gaps that shape every filter written against them.

## Anti-triggers

- **The same machine in the RMM** — ConnectWise Automate calls it a
  `Computer` and keeps its own ID space; an Automate `ComputerID` never
  matches a PSA `configuration/id`. Use `connectwise-automate-computers`.
- **A product on a quote or in the catalog** — a sellable item is a catalog
  product, not something the client already owns; use
  `connectwise-psa-product-catalog`.
- **Network devices discovered by a monitoring tool** — a discovered device
  is not a PSA record until somebody creates one; use `auvik-devices`.
- **Documentation about an asset** — credentials, diagrams and runbooks live
  in the documentation platform; use `hudu-assets`.
- **What the client is contracted for** — coverage is an agreement question,
  not an asset question; use `connectwise-psa-agreements`.

## Tool surface

Configurations are reached through `connectwise-manage-mcp`, never by direct
REST. The endpoint below is named for orientation only.

```
Base: /company/configurations
```

| Tool | Purpose |
|------|---------|
| `cw_search_configurations` | Search with CW conditions syntax |
| `cw_get_configuration` | Fetch one configuration by ID |

Both are read-tier. **There is no tool that creates, updates, or deletes a configuration**, and none that enumerates configuration types or statuses — see [Not available through the tool surface](#not-available-through-the-tool-surface).

## Searching

```
cw_search_configurations
  conditions: "company/id=12345 and status/name=\"Active\""
  orderBy:    "name asc"
  pageSize:   25
```

```
cw_get_configuration  id: 4471
```

Common condition fields:

| Field | Matches |
|-------|---------|
| `company/id` | Exact company, once resolved with `cw_search_companies` |
| `name` | The configuration's display name; supports `contains` |
| `serialNumber` | Manufacturer serial, exact |
| `tagNumber` | Asset tag the MSP assigned, exact |
| `type/name` | Configuration type, **by name** |
| `status/name` | Configuration status, **by name** |
| `site/name` | Physical location within the company |

## Types and statuses filter by name, and cannot be validated first

**No tool enumerates configuration types, and none enumerates configuration statuses.** The consequence is specific and easy to get wrong: a type or status the caller supplies cannot be checked before it is used, so it goes straight into the conditions string as a name match.

That makes one failure mode ambiguous. An empty result means *no match for that filter* **or** *that filter names nothing in this instance*, and without the lists there is no way to tell them apart. Report the empty result as the former and say the list could not be validated — never as *this client owns no configurations*, which is a different and probably false claim.

The tables below are **retained as domain knowledge rather than a validated list**. They record what these values typically look like; any given instance may differ.

| Typical type | Usually describes |
|--------------|-------------------|
| Server | Physical or virtual server |
| Workstation | Desktop or laptop |
| Network Device | Switch, router, access point |
| Firewall | Perimeter device |
| Printer | Print device |
| Managed Service | A recurring service treated as an asset |
| Licence | A software entitlement |
| Backup | A backup target or job |

| Typical status | Usually means |
|----------------|---------------|
| Active | In service |
| Inactive | Retired but retained for history |
| In Stock | Owned, not deployed |
| RMA | Out for replacement |

## What a configuration record carries

A configuration returned by `cw_get_configuration` typically includes its name, type, status, company, site, contact, serial and tag numbers, purchase and warranty dates, vendor, and any instance-specific questions the PSA has been configured to ask. **Warranty and purchase dates are readable and not settable here**, which makes this surface useful for a warranty-expiry answer and useless for correcting one.

## Gotchas

- **A configuration belongs to a company, and optionally to a site.** A company with several locations puts the same device type at each, so a dispatch answer that omits `site` is incomplete. Note that site is readable on the configuration while the company's site list is not enumerable — see [ConnectWise Companies](../companies/SKILL.md).
- **Serial numbers are not unique across manufacturers.** Filter on `serialNumber` together with `company/id` unless a global search is genuinely intended.
- **Inactive configurations stay in search results.** Add `status/name` to any query that means *currently in service*; the API does not assume it.
- **A ticket and a configuration are related in the PSA and not reachable from each other here.** Neither `cw_get_ticket` nor `cw_search_configurations` traverses the association, so *"which tickets were raised against this server"* cannot be answered on this surface.

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Enumerating configuration types | A configuration-type tool |
| Enumerating configuration statuses | A configuration-status tool |
| Tickets related to a configuration | A tool exposing the ticket-configuration association. **Neither side is queryable by the other** |
| Creating, updating, or deleting a configuration | A write tool. **The surface exposes none** — both configuration tools here are read-tier |

**The two enumeration gaps are the reason a filter cannot be validated**, and they are per-tenant configuration rather than fixed vocabulary — which is exactly why hardcoding an id from another instance is drift rather than a shortcut.

**The ticket association gap is a real relationship the surface omits.** A ticket in the PSA can name the configurations it concerns, and that link is reachable from neither direction here, so an asset history assembled on this surface is assembled from ticket text rather than from the association.

## Related Skills

- [ConnectWise Companies](../companies/SKILL.md) — resolving the owning company
- [ConnectWise Tickets](../tickets/SKILL.md) — the service side, and the association neither can traverse
- [ConnectWise Agreements](../agreements/SKILL.md) — what the client is contracted for
- [ConnectWise Product Catalog](../product-catalog/SKILL.md) — sellable items, as distinct from owned ones
- [ConnectWise API Patterns](../api-patterns/SKILL.md) — conditions syntax and paging
