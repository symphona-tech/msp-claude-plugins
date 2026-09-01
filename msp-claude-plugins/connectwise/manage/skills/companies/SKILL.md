---
name: "ConnectWise Manage Companies"
description: >
  ConnectWise PSA company/account management: company types, statuses,
  sites/locations, custom fields, and company relationships. Essential for MSP
  account management and CRM operations in ConnectWise PSA.
when_to_use: >-
  When creating, updating, searching, or managing company/account records. Use when: connectwise
  company, connectwise account, company management, create company connectwise, company site,
  company location, company type, company status, customer record, client record, or company
  custom field.
---

# ConnectWise PSA Company Management

## Overview

Companies in ConnectWise PSA represent your clients, prospects, vendors, and other business entities. Company records are central to ticketing, agreements, projects, and billing. This skill covers company CRUD operations, types, statuses, sites, and custom fields.

## Anti-triggers

- **The same customer in Automate** — ConnectWise Automate calls it a
  `Client` and keeps its own ID space; a PSA `company/id` never matches an
  Automate `ClientID`. Use `connectwise-automate-clients`.
- **People at the company** — names, email addresses, phone numbers and
  portal logins are contact records; use `connectwise-psa-contacts`.
- **The same client in a documentation or distributor platform** — a
  Hudu company is a documentation container and a Pax8 company is the
  billing account licences are bought against; neither shares an ID
  with the PSA. Use `hudu-companies` or `pax8-companies`.

## Tool surface

Companies are reached through `connectwise-manage-mcp`, never by direct REST.
The endpoint below is named for orientation only — it is not a route this
plugin calls.

```
Base: /company/companies
```

| Tool | Purpose |
|------|---------|
| `cw_search_companies` | Search with CW conditions syntax |
| `cw_get_company` | Fetch one company by ID |
| `cw_create_company` | Create a company |
| `cw_update_company` | JSON Patch update |

There is **no delete tool**, and no tool for company sites or custom fields —
see [Not available through the tool surface](#not-available-through-the-tool-surface).

## Company Types

Standard company types in ConnectWise PSA:

| Type ID | Name | Description |
|---------|------|-------------|
| 1 | Client | Active paying customer |
| 2 | Prospect | Potential customer |
| 3 | Vendor | Supplier or partner |
| 4 | Partner | Strategic partner |
| 5 | Competitor | Market competitor |

**Note:** Company types are configurable. Query `/company/companies/types` for your instance's types.

## Company Statuses

Standard company statuses:

| Status ID | Name | Description | Active |
|-----------|------|-------------|--------|
| 1 | Active | Active company | Yes |
| 2 | Inactive | Inactive company | No |
| 3 | Not Approved | Pending approval | No |

Query `/company/companies/statuses` for available statuses.

## Complete Company Field Reference

### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | int | System | Auto-generated unique identifier |
| `identifier` | string(25) | Yes | Unique company code (e.g., "ACME") |
| `name` | string(50) | Yes | Full company name |
| `status` | object | No | `{id: statusId}` |
| `type` | object | No | `{id: typeId}` |

### Contact Information

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phoneNumber` | string(30) | No | Main phone |
| `faxNumber` | string(30) | No | Fax number |
| `website` | string(255) | No | Company website URL |

### Address Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `addressLine1` | string(50) | No | Street address |
| `addressLine2` | string(50) | No | Suite/unit |
| `city` | string(50) | No | City |
| `state` | string(50) | No | State/province |
| `zip` | string(12) | No | Postal code |
| `country` | object | No | `{id: countryId}` |

### Classification Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `territory` | object | No | `{id: territoryId}` - Sales territory |
| `market` | object | No | `{id: marketId}` - Industry/market |
| `accountNumber` | string(30) | No | External accounting ID |
| `taxIdentifier` | string(25) | No | Tax ID/EIN |
| `annualRevenue` | decimal | No | Company annual revenue |
| `numberOfEmployees` | int | No | Employee count |

### Billing Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `billingTerms` | object | No | `{id: termsId}` - Payment terms |
| `billToCompany` | object | No | `{id: companyId}` - Bill to different company |
| `invoiceDeliveryMethod` | object | No | `{id: methodId}` - Email, Mail, etc. |
| `invoiceTemplate` | object | No | `{id: templateId}` |
| `pricingSchedule` | object | No | `{id: scheduleId}` |

### Ownership Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ownerLevel` | object | No | `{id: levelId}` - Account manager level |
| `defaultContact` | object | No | `{id: contactId}` - Primary contact |
| `leadSource` | string(50) | No | How lead was acquired |
| `leadFlag` | boolean | No | Is this a lead |

### Tracking Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dateAcquired` | date | No | When became customer |
| `deletedFlag` | boolean | System | Soft delete status |
| `mobileGuid` | guid | System | Mobile app identifier |
| `_info` | object | System | Metadata including last updated |

## Company Sites

Sites represent physical locations for a company. Each company can have multiple sites.

### Site Endpoint

```
/company/companies/{companyId}/sites
```

### Site Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | int | System | Site identifier |
| `name` | string(50) | Yes | Site name |
| `addressLine1` | string(50) | No | Street address |
| `addressLine2` | string(50) | No | Suite/unit |
| `city` | string(50) | No | City |
| `state` | string(50) | No | State/province |
| `zip` | string(12) | No | Postal code |
| `country` | object | No | `{id: countryId}` |
| `phoneNumber` | string(30) | No | Site phone |
| `faxNumber` | string(30) | No | Site fax |
| `taxCode` | object | No | `{id: taxCodeId}` |
| `defaultFlag` | boolean | No | Is primary site |

### Site structure

**No tool creates or reads sites.** The shape below documents how ConnectWise
models a site, for reading a company record that references one:

```json
{
  "name": "Main Office",
  "addressLine1": "123 Main St",
  "city": "Springfield",
  "state": "IL",
  "zip": "62701",
  "primaryAddressFlag": true
}
```

## Custom Fields

Custom fields store company-specific data not in standard fields.

### Custom field structure

**No tool reads or writes custom fields.** They appear on a company record as a
`customFields` array when the record is fetched with `cw_get_company`, and are
read-only from this plugin's perspective.

### Custom Field Response

```json
{
  "id": 1,
  "caption": "SLA Tier",
  "value": "Gold",
  "type": "Text"
}
```

### Custom field values

Custom field values cannot be set through the tool surface. `cw_update_company`
takes JSON Patch operations against the company record; whether a given custom
field is reachable that way is **not established** by the tool's schema, and
guessing a patch path against a live company record is not worth the risk.

### Custom Field Types

| Type | Description |
|------|-------------|
| `Text` | Free-form text |
| `Number` | Numeric value |
| `Date` | Date value |
| `Checkbox` | Boolean true/false |
| `Dropdown` | Selection from list |

## Tool Operations

### Search Companies

```
cw_search_companies
  conditions: "status/name=\"Active\" and territory/id=1"
  orderBy:    "name asc"
  pageSize:   100
```

### Get a Company

```
cw_get_company  id: 12345
```

To find one by identifier rather than ID, search for it:

```
cw_search_companies  conditions: "identifier=\"ACME\""
```

### Create a Company

```
cw_create_company
  name:       "Acme Corporation"
  identifier: "ACME"
```

Check the tool's input schema for the fields it accepts. The field reference
above documents the full ConnectWise model, which is **wider than the tool's
arguments** — a field appearing there is not a guarantee the tool can set it.

### Update a Company

```
cw_update_company
  id: 12345
  operations: [
    {"op": "replace", "path": "status/id", "value": 2},
    {"op": "replace", "path": "phoneNumber", "value": "555-0100"}
  ]
```

`replace` overwrites. To append to a field, read it with `cw_get_company` first
and send the concatenated value.

## Common Query Patterns

**Active clients:**
```
conditions=status/id=1 and type/id=1
```

**Companies by territory:**
```
conditions=territory/id=5
```

**Companies with no contact:**
```
conditions=defaultContact=null
```

**Recently added companies:**
```
conditions=_info/lastUpdated>[2024-01-01]
orderBy=_info/lastUpdated desc
```

**Search by name:**
```
conditions=name contains "tech"
```

**Companies by market:**
```
conditions=market/id=3
```

## Company Relationships

### Parent/Child Companies

Companies can have hierarchical relationships. A child company carries a
`parentCompany` reference on its own record, so the hierarchy is readable from
`cw_get_company` without a separate call. There is no tool for the
`managedDevicesIntegrations` collection.

### Related Entities

| Entity | Relationship |
|--------|-------------|
| Contacts | `/company/contacts?conditions=company/id={id}` |
| Tickets | `/service/tickets?conditions=company/id={id}` |
| Agreements | `/finance/agreements?conditions=company/id={id}` |
| Projects | `/project/projects?conditions=company/id={id}` |
| Configurations | `/company/configurations?conditions=company/id={id}` |

## Best Practices

1. **Use unique identifiers** - Keep short, meaningful codes (ACME, ABC123)
2. **Standardize company names** - Consistent naming helps searching
3. **Set company type** - Enables filtering and reporting
4. **Add default contact** - Primary point of contact for communications
5. **Configure sites** - Multiple locations need separate sites
6. **Use custom fields** - Store industry-specific data
7. **Link to accounting** - Set accountNumber for integration

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Identifier required | Missing identifier | Provide unique company code |
| Name required | Missing company name | Include name field |
| Identifier exists | Duplicate identifier | Choose unique identifier |
| Cannot delete | Has related records | Set status to Inactive instead |
| Invalid status | Status doesn't exist | Query statuses endpoint |

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Deleting a company | A delete tool. **The surface exposes none** — `GOVERNANCE.md` records the Delete group as empty |
| Company sites (`/company/companies/{id}/sites`) | A sites tool |
| Custom field read or update | A custom-fields tool |

The sites and custom-field sections above are retained as **domain knowledge**:
they describe how ConnectWise models this data, which is useful when reading a
company record. **They are not workflows this plugin can execute.**

Company deletion is not merely unavailable here — it is absent from the whole
inventoried surface, so no amount of configuration exposes it.

## Related Skills