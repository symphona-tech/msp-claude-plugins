---
name: "Hudu Companies"
description: >
  Hudu companies (clients/organizations): company field reference, parent/child
  hierarchy, PSA integration matching via id_in_integration, the hudu-mcp company
  tools (list, get, create, update, archive, unarchive, delete), onboarding and
  offboarding workflows, and how companies scope assets, passwords, and articles.
when_to_use: >-
  When looking up, creating, updating, or archiving a company in Hudu, or scoping other
  Hudu records to a client. Use when: hudu
  company, hudu client, hudu organization, company lookup, company documentation, company
  management, hudu org, or client documentation.
---

# Hudu Companies Management

## Overview

Companies are the foundational entity in Hudu, representing clients, vendors, or internal entities. All documentation, assets, passwords, articles, and websites are associated with a company. In Hudu, the "Company" label is customizable per instance -- some MSPs rename it to "Organization" or "Client" -- but the tools are always the `hudu_*_company` / `hudu_list_companies` tools below.

## Anti-triggers

- **The client record of record** — a Hudu company scopes documentation.
  Contracts, contacts, and service history live in the PSA; use
  `autotask-crm`, `connectwise-psa-companies`, or `halopsa-clients`.
  Hudu's `id_in_integration` field holds the PSA's ID precisely because
  the two are different records.
- **The same client in IT Glue** — IT Glue calls these Organizations; use
  `itglue-api-patterns` for its equivalent surface.
- **The monitored client container** — an RMM organization or site is a
  monitoring scope, not documentation; use `ninjaone-organizations`,
  `atera-customers`, or `datto-rmm-sites`.
- **The client's licence or billing entity** — use `pax8-companies`,
  `sherweb-customers`, or `qbo-customers`.

## Key Concepts

### Company Types

Unlike IT Glue, Hudu does not enforce built-in company types. Companies are typically organized using custom fields or naming conventions. Common patterns MSPs use:

| Pattern | Description | Example |
|---------|-------------|---------|
| Active Client | Currently serviced customer | Standard operational state |
| Prospect | Potential client | Pre-sales documentation |
| Vendor | Product/service supplier | Software vendors |
| Internal | Your own MSP | Internal documentation |
| Former Client | Previously serviced | Historical records |

### Company Hierarchy

Companies can have parent/child relationships for multi-location or multi-division clients:

```
Parent Company (Acme Holdings)
+-- Child: Acme East Division
+-- Child: Acme West Division
+-- Child: Acme International
```

### PSA Integration

Companies can be matched to PSA records using the `id_in_integration` and `integration_slug` fields, enabling cross-platform lookups between Hudu and tools like ConnectWise Manage, Autotask, or HaloPSA.

## Field Reference

### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | integer | System | Auto-generated unique identifier |
| `name` | string | Yes | Company name |
| `nickname` | string | No | Short name or abbreviation |
| `company_type` | string | No | Type classification |
| `address_line_1` | string | No | Street address line 1 |
| `address_line_2` | string | No | Street address line 2 |
| `city` | string | No | City |
| `state` | string | No | State/province |
| `zip` | string | No | Postal code |
| `country_name` | string | No | Country |
| `phone_number` | string | No | Phone number |
| `fax_number` | string | No | Fax number |
| `website` | string | No | Company website URL |
| `id_number` | string | No | Free-form company ID number (filterable on `hudu_list_companies`) |
| `notes` | string | No | Rich text notes |

### Integration Fields

| Field | Type | Description |
|-------|------|-------------|
| `id_in_integration` | integer | PSA system company ID |
| `integration_slug` | string | PSA integration identifier |

Both integration fields are written by Hudu's own PSA integration. They are readable on the returned company record, but no tool sets them and no list tool filters on them.

### Relationship Fields

| Field | Type | Description |
|-------|------|-------------|
| `parent_company_id` | integer | Parent company ID |
| `parent_company_name` | string | Parent company name (read-only) |

### Metadata Fields

| Field | Type | Description |
|-------|------|-------------|
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last update timestamp |
| `slug` | string | URL-friendly identifier |
| `object_type` | string | Always "Company" |

## Tools

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

| Tool | What it does |
|------|--------------|
| `hudu_list_companies` | List companies, one page at a time, with optional filters |
| `hudu_get_company` | Get one company by id |
| `hudu_create_company` | Create a company |
| `hudu_update_company` | Update the fields you pass on an existing company |
| `hudu_archive_company` | Archive a company (reversible) |
| `hudu_unarchive_company` | Restore an archived company |
| `hudu_delete_company` | Permanently delete a company and everything scoped to it |

### List Companies

`hudu_list_companies` takes optional `name`, `city`, `state`, `phone_number`, `website`, `id_number`, `archived`, `page` and `page_size`.

```json
{ "name": "Acme", "archived": false, "page": 1, "page_size": 25 }
```

The tool returns one page: default `page_size` 25, `page` is 1-indexed, and the result carries no total count — its "Found N" message counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none. Reporting the first page as the whole result is the most plausible wrong answer here.

There is no free-text `search` argument and no `id_in_integration` filter. To find a company by PSA id, page through `hudu_list_companies` and match `id_in_integration` on the returned records. The same applies to `parent_company_id`: finding a parent's children means paging every company and matching the field.

### Get Single Company

`hudu_get_company` takes `id` (required).

### Create Company

`hudu_create_company` takes `name` (required) and optional `nickname`, `company_type`, `address_line_1`, `address_line_2`, `city`, `state`, `zip`, `country_name`, `phone_number`, `fax_number`, `website`, `id_number`, `notes` and `parent_company_id`.

```json
{
  "name": "New Client Corporation",
  "nickname": "NCC",
  "company_type": "Customer",
  "address_line_1": "123 Main Street",
  "city": "Portland",
  "state": "OR",
  "zip": "97201",
  "phone_number": "555-123-4567",
  "website": "https://newclient.com",
  "notes": "Onboarded February 2026. Primary contact: John Smith."
}
```

### Update Company

`hudu_update_company` takes `id` (required) and the same optional fields as create. Only the fields you pass are sent.

```json
{
  "id": 123,
  "nickname": "NCC-UPDATED",
  "notes": "Updated: New primary contact is Jane Doe (555-987-6543)."
}
```

### Delete Company

`hudu_delete_company` takes `id` (required).

**Warning:** Deletion is irreversible. Deleting a company deletes its assets, articles and passwords with it. Deletion is a per-API-key permission in Hudu; if it is disabled on the server's key, the tool returns an error rather than deleting.

### Archive / Unarchive Company

`hudu_archive_company` and `hudu_unarchive_company` each take `id` (required). Company archiving is reversible: `hudu_unarchive_company` restores the company. This is the only unarchive tool — archived assets and articles have no tool to restore them.

## Common Workflows

### New Client Onboarding

1. **Create company** with basic info (name, address, phone, website) using `hudu_create_company`
2. **Link to PSA** — the `id_in_integration` link is set by Hudu's PSA integration when it syncs the company; no tool here sets it
3. **Add notes** for quick reference (primary contact, contract info) in the `notes` argument
4. **Create initial assets** (servers, workstations, network devices)
5. **Document passwords** for the company
6. **Create articles** (network overview, procedures)

### Client Offboarding

1. **Review** critical documentation with the read tools; exporting it is a Hudu web UI action
2. **Leave password records in place** (do not delete for audit purposes); no tool archives a password
3. **Add offboarding notes** with date and reason: `hudu_update_company` with `id` and `notes`
4. **Archive the company** instead of deleting: `hudu_archive_company` with `id`. It can be restored later with `hudu_unarchive_company`

### PSA Sync Verification

1. Page through `hudu_list_companies` until a page returns fewer than `page_size` items.
2. Split the companies into those with `id_in_integration` set and those without.
3. Checking that each linked id still exists in the PSA is done with that PSA's own plugin; this plugin reads only Hudu.

### Bulk Company Report

Page through `hudu_list_companies` to exhaustion and report, per company: `name`, `nickname`, `city`, `state`, whether `id_in_integration` is set, whether `website` is set, `created_at` and `updated_at`.

## Failure modes

Errors reach the agent as a tool error string, not an HTTP status.

| Condition | What the tool returns | Resolution |
|-----------|-----------------------|------------|
| `name` missing, duplicate name, or invalid value | `Validation error` | Provide a unique `name`; on a duplicate, find the existing company with `hudu_list_companies` and `name` |
| `parent_company_id` does not exist | `Validation error` | Verify the parent with `hudu_get_company` |
| Company id does not exist or was deleted | `Resource not found` | Re-find the company with `hudu_list_companies` |
| Deletion disabled on the server's API key, or other permission gap | `Access forbidden - insufficient permissions` or another tool error | Archive instead, or ask a Hudu administrator |
| Every tool fails authentication | `Authentication failed - invalid API key` | The MCP server's key is the problem; report it to whoever operates the server |
| Too many requests | `Rate limit exceeded and max retries reached` | The server already retried; wait before retrying |
| Hudu server fault | `Server error: <status>` | The server already retried once; retry later |

## Best Practices

1. **Use descriptive names** - Include location or identifier if needed for uniqueness
2. **Set nicknames** - Short abbreviations for quick reference
3. **Maintain notes** - Keep emergency contact info and contract details readily available
4. **Link to PSA** - Enable Hudu's PSA integration so `id_in_integration` is set for cross-platform lookups
5. **Use parent/child** - Organize multi-location or division clients
6. **Archive, don't delete** - Preserve historical documentation
7. **Include address info** - Useful for dispatch and site visit planning
8. **Document website** - Track the company's primary website URL

## Related Skills

- [Hudu Assets](../assets/SKILL.md) - Asset management for companies
- [Hudu Articles](../articles/SKILL.md) - Knowledge base articles
- [Hudu Passwords](../passwords/SKILL.md) - Credential storage
