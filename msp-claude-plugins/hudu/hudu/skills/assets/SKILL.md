---
name: "Hudu Assets"
description: >
  Hudu assets and asset layouts through the hudu-mcp tools: the layout-as-template model, custom field types, writing `custom_fields` keyed by the layout's field labels, one-way archiving vs deletion, company scoping, and the filter and paging patterns of hudu_list_assets and hudu_list_asset_layouts.
when_to_use: >-
  When creating, querying, updating, or archiving documented items in Hudu, or when designing
  the asset layouts that define their fields. Use when: hudu asset, hudu configuration, hudu
  server, hudu workstation, hudu device, asset layout, asset management, device inventory,
  hardware tracking, or hudu ci.
---

# Hudu Assets Management

## Overview

Assets in Hudu represent documented items such as servers, workstations, network devices, applications, and any other infrastructure or service an MSP needs to track. Unlike some platforms with fixed asset types, Hudu uses **asset layouts** -- customizable templates that define the fields and structure for each type of asset. This means your Hudu instance might have asset layouts for "Server," "Workstation," "Firewall," "Microsoft 365 Tenant," or any custom type your team defines.

## Anti-triggers

- **The live state of a machine** — a Hudu asset is a documentation
  record. It does not know whether the server is online, patched, or
  alerting, and it goes stale silently. For the running endpoint use
  `ninjaone-devices`, `atera-devices`, `datto-rmm-devices`,
  `ncentral-devices`, or `connectwise-automate-computers`.
- **The same record in IT Glue** — IT Glue calls these Configurations and
  Flexible Assets; use `itglue-configurations` or
  `itglue-flexible-assets`. The two platforms model custom fields
  differently, so the field shapes do not transfer.
- **A credential attached to the asset** — passwords are separate tools with their own permission model, linked to the asset by `passwordable_type` and `passwordable_id` when the password is created; use `hudu-passwords`.
- **A runbook or procedure about the asset** — prose documentation is
  `hudu-articles`.
- **What changed on the system recently** — configuration drift is
  detected elsewhere; use `liongard-detections`.
- **The asset row in a PSA or RMM rather than the documentation
  platform** — those records carry contract, ticket, and agent state
  that Hudu never sees, and they are keyed independently; use
  `halopsa-assets`, `syncro-assets`, or `superops-assets`.

## Key Concepts

### Asset Layouts

Asset layouts are templates that define what fields an asset of that type contains. Each layout has:

- A name (e.g., "Server," "Workstation," "Network Device")
- A set of custom fields with types (text, rich text, number, date, checkbox, dropdown, etc.)
- An icon and color for visual identification
- Optional: whether it appears in the sidebar, its position, etc.

Common asset layouts in MSP environments:

| Layout | Description | Typical Fields |
|--------|-------------|----------------|
| Server | Physical or virtual servers | Hostname, IP, OS, RAM, CPU, serial |
| Workstation | End-user devices | Hostname, user, OS, serial, warranty |
| Network Device | Routers, switches, firewalls | IP, model, firmware, port count |
| Printer | Print devices | IP, model, serial, location |
| Application | Software/services | Version, license key, vendor |
| Microsoft 365 | M365 tenant details | Tenant ID, domain, license count |
| Backup | Backup configuration | Solution, server, schedule, retention |

### Custom Fields

Each asset layout defines custom fields. Field types include:

| Type | Description | Example |
|------|-------------|---------|
| Text | Single-line text | Hostname, serial number |
| RichText | HTML rich text | Notes, description |
| Number | Numeric value | RAM (GB), port count |
| Date | Date value | Warranty expiry, install date |
| CheckBox | Boolean | Monitored (yes/no) |
| Dropdown | Predefined options | OS type, status |
| Email | Email address | Admin contact |
| Phone | Phone number | Support line |
| Password | Embedded password | Admin credentials |
| AssetTag | Link to another asset | Host server, parent device |
| Website | URL | Management portal |

### Asset vs Asset Layout

- **Asset Layout** = the template/schema (like a database table definition)
- **Asset** = an instance of a layout (like a row in the table)

### Fields

Every asset requires `company_id`, `asset_layout_id`, and `name`. `primary_serial`, `primary_model`, and `primary_mail` are first-class columns; everything else lives in the layout-defined custom fields.

See [references/fields.md](references/fields.md) for the complete field reference, including asset layout fields.

## Tools

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

| Tool | What it does |
|------|--------------|
| `hudu_list_assets` | List assets, one page at a time, filtered by company, layout, name, serial or archived state |
| `hudu_get_asset` | Get one asset by id, including its `fields` |
| `hudu_create_asset` | Create an asset under a company and layout |
| `hudu_update_asset` | Update an asset; only the fields given are sent |
| `hudu_archive_asset` | Archive an asset — one-way through this plugin |
| `hudu_delete_asset` | Delete an asset permanently |
| `hudu_list_asset_layouts` | List asset layouts, optionally filtered by name |
| `hudu_get_asset_layout` | Get one layout, including its field definitions |
| `hudu_create_asset_layout` | Create an asset layout |
| `hudu_update_asset_layout` | Update an asset layout |

`hudu_list_assets` takes optional `company_id`, `asset_layout_id`, `name`, `primary_serial`, `archived`, `page` and `page_size`. It returns one page — default `page_size` 25, `page` 1-indexed — and no total count; its "Found N" message counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none. Reporting the first page as the whole result is the most plausible wrong answer here.

`hudu_create_asset` takes `company_id`, `asset_layout_id` and `name` (all required), and optional `custom_fields`, `primary_mail`, `primary_manufacturer`, `primary_model` and `primary_serial`. `hudu_update_asset` takes `id` and any of the same fields.

`custom_fields` is an object with the layout's field labels as keys. Read the layout with `hudu_get_asset_layout` first to learn its field labels, types and which are required — do not invent keys. On read, the values come back on the asset as `fields`, not `custom_fields`.

`hudu_archive_asset` takes `id`. Its tool title says "(reversible)", but no tool unarchives an asset: restoring one is a manual action in the Hudu web UI. Do not archive expecting to undo it. `hudu_delete_asset` takes `id` and is irreversible.

`hudu_list_asset_layouts` takes optional `name`, `page` and `page_size`; it has been observed to ignore `page_size`, so stop paging on an empty page. `hudu_get_asset_layout` takes `id`.

See [references/api.md](references/api.md) for the complete tool reference with argument examples.

## Common Workflows

### Asset Onboarding

Layout IDs are instance-specific — resolve the layout by name before creating the asset rather than hardcoding an ID.

1. Call `hudu_list_asset_layouts` with `name` set to the layout name (for example `Server`) and take its `id`. If no layout matches, stop and report it.
2. Call `hudu_get_asset_layout` with that `id` and read `fields` for the labels and which are `required`.
3. Call `hudu_create_asset` with `company_id`, `asset_layout_id`, `name`, `primary_serial`, `primary_model`, and `custom_fields` keyed by those labels, supplying every required field.

### Warranty Tracking

Warranty dates live in a layout-defined custom field, so no tool filters on them — page through the assets and filter each record's `fields` yourself.

1. Call `hudu_list_assets` with `archived: false` (and `asset_layout_id` or `company_id` to narrow it), requesting `page` 1, 2, 3… until a page returns fewer items than `page_size`.
2. On each asset, read the warranty value from `fields`, keep those falling within the window, and sort by date.

### Asset Decommissioning

1. Call `hudu_update_asset` with the asset `id` and `custom_fields` setting the layout's notes field to a decommission note with the date and reason.
2. Call `hudu_archive_asset` with the `id`. This is one-way through this plugin — restoring the asset is a manual action in the Hudu web UI — so confirm before archiving.

### Asset Inventory by Company

1. Call `hudu_list_assets` with `company_id` and `archived: false`, paging until a page returns fewer items than `page_size`.
2. Group the assets by `asset_layout_id`, resolving layout names with `hudu_list_asset_layouts`, and report each asset's `name`, `primary_serial`, `primary_model` and `updated_at`.

## Gotchas

- **`custom_fields` on write, `fields` on read.** Round-tripping an asset requires renaming the key.
- **Custom fields are not queryable.** Only `company_id`, `asset_layout_id`, `name`, `primary_serial`, and `archived` filter in `hudu_list_assets`; anything layout-defined must be filtered after paging through the results.
- **A `Validation error` on create usually means a layout-required field is missing.** Call `hudu_get_asset_layout` and inspect `fields` where `required: true` — the error does not name the field.
- **Layout IDs differ per Hudu instance.** Look them up by name; never hardcode.
- **Archive is a distinct tool** (`hudu_archive_asset`), not an `archived` field on update, and it is one-way through this plugin: no tool unarchives an asset, despite the tool title saying "(reversible)". Restoring one is a manual action in the Hudu web UI.
- **Relations have no filter.** `hudu_list_relations` takes only `page` and `page_size`; finding an asset's relations means paging through all relations and matching the asset's id against `fromable_id` / `toable_id`. No tool creates a relation.
- **Passwords linked to an asset** are created with `hudu_create_asset_password`, setting `passwordable_type` to `Asset` and `passwordable_id` to the asset's id; only create sets the link. See `hudu-passwords`.

See [references/errors.md](references/errors.md) for the complete error and validation table plus a recovery pattern.

## Best Practices

1. **Standardize naming** - Use consistent format (e.g., SITE-TYPE-NUM: NYC-DC-01)
2. **Use appropriate layouts** - Choose the right asset layout for the device type
3. **Track serial numbers** - Enable warranty lookups and asset verification
4. **Document custom fields** - Fill in all relevant fields, not just the name
5. **Archive, don't delete** - Preserve historical records for decommissioned assets; archiving is one-way through this plugin, so only archive what is really retired
6. **Create layouts thoughtfully** - Design layouts with fields MSP technicians actually need
7. **Keep layouts consistent** - Use the same layout across all companies for the same device type
8. **Link related assets** - Use AssetTag fields to connect VMs to hosts, apps to servers

## Related Skills

- [Hudu Companies](../companies/SKILL.md) - Parent company management
- [Hudu Passwords](../passwords/SKILL.md) - Device credentials
- [Hudu Articles](../articles/SKILL.md) - Device documentation
