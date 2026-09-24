---
description: Find a company in Hudu by name
argument-hint: "<name> [status]"
arguments: [name, status]
---

# Find Hudu Company

Find a company in Hudu by name with optional filters.

The Hudu MCP server must be connected.

## Steps

1. **Parse search parameters**
   - Extract company name query
   - Set status filter

2. **Execute search**
   - Search companies by name (partial match) with `hudu_list_companies` and `name`
   - Apply status filter through the `archived` argument
   - Page until a page returns fewer items than `page_size`; "Found N" in the tool's message counts one page only
   - For a single match, fetch related summary data (resource counts, parent and children)

3. **Format and return results**
   - Display company details
   - Include resource counts
   - Provide quick actions

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| name | string | Yes | - | Company name (partial match) |
| status | string | No | active | Status filter (active/archived/all) |

## Examples

### Basic Search

```
/find-company "Acme"
```

### Search All Statuses

```
/find-company "Acme" --status all
```

### Search Archived

```
/find-company "Corp" --status archived
```

## Output

### Single Match

```
Found 1 company matching "Acme Corporation"

Company: Acme Corporation
================================================================
ID:            123
Nickname:      ACME
Type:          Customer
PSA ID:        12345

Address:
  123 Main Street
  Springfield, IL 62704

Phone:         555-123-4567
Website:       https://www.acme.com

Notes:
Primary contact: John Smith (555-123-4567)
Contract renewal: March 2026

Resources:
================================================================
Assets:            42 items
Passwords:         28 items
Articles:          12 items

Parent Company: Acme Holdings (ID: 100)

Child Companies:
  - Acme East Division (ID: 124)
  - Acme West Division (ID: 125)

Created: 2020-06-15
Updated: 2026-02-10

Quick Actions:
  - View assets: /lookup-asset --company "Acme Corporation"
  - Search articles: /search-articles --company "Acme Corporation"
  - Get password: /get-password --company "Acme Corporation"
  - Open in Hudu: [link]
================================================================
```

### Multiple Matches

```
Found 4 companies matching "Acme"

+------------------------+----------+----------+--------+--------+---------+
| Name                   | Nickname | Status   | Assets | Articles | PSA ID |
+------------------------+----------+----------+--------+--------+---------+
| Acme Corporation       | ACME     | Active   | 42     | 12     | 12345   |
| Acme East Division     | ACME-E   | Active   | 18     | 5      | 12346   |
| Acme West Division     | ACME-W   | Active   | 24     | 7      | 12347   |
| Acme Holdings          | ACME-H   | Active   | 3      | 2      | 12340   |
+------------------------+----------+----------+--------+--------+---------+

Select company:
  /find-company "Acme Corporation"
  /find-company "Acme East"
```

### No Results

```
No companies found matching "XYZ Company"

Suggestions:
  - Check spelling of the company name
  - Try a partial name match
  - Include archived: --status all
  - Try different keywords

Example searches:
  /find-company "XYZ"
  /find-company "Company" --status all
```

### Archived Companies

```
/find-company "Old Client" --status archived

Found 1 archived company matching "Old Client"

Company: Old Client Inc (ARCHIVED)
================================================================
ID:            1000
Nickname:      OCI
Type:          Former Client

Notes:
Client offboarded 2024-06-15. Final contact: Bob Manager.
Data retained for compliance.

Resources (historical):
================================================================
Assets:            15 items (archived)
Passwords:         12 items
Articles:          6 items

This company is archived. Restore it with hudu_unarchive_company if needed.
================================================================
```

## Status Values

| Value | Description |
|-------|-------------|
| active | Currently active companies (default) |
| archived | Archived/historical companies |
| all | All companies regardless of status |

## Tools

Called in this order:

1. `hudu_list_companies` — `name` set to the query, `archived: false` for `active`, `archived: true` for `archived`, and one call of each merged for `all`; request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none.
2. For a single match only: `hudu_list_assets`, `hudu_list_articles` and `hudu_list_asset_passwords`, each with `company_id`, paged to exhaustion and counted. No tool returns a count directly, so a count is the number of records paged. If `hudu_list_asset_passwords` returns `Authentication failed - invalid API key` while the other tools work, the server's API key has password access disabled: show `Passwords: not measured — credential access not enabled`, never a count of zero, and never report the key as invalid.
3. For a single match only: parent company from `parent_company_id` / `parent_company_name` on the record; child companies by paging `hudu_list_companies` and matching `parent_company_id` to this company's `id`. This pages every company, so skip it when the match list is long.

Website records are not read — the website tools return no data at this server version. The company's own `website` field is shown as recorded.

## Error Handling

### No Results

```
No companies found matching "xyz"

Suggestions:
  - Check spelling of the company name
  - Try a partial match
  - Include all statuses: --status all

Example:
  /find-company "xyz" --status all
```

### Invalid Status

```
Invalid status: "deleted"

Valid statuses:
  - active
  - archived
  - all

Example:
  /find-company "Acme" --status archived
```

### Tool Error

```
Hudu tool call failed: Authentication failed - invalid API key

Possible causes:
  - Every tool fails this way: the MCP server's Hudu API key is invalid; report it to whoever operates the server
  - Only password tools fail this way: the key has password access disabled; counts are shown as not measured
  - Rate limit exceeded and max retries reached: wait before retrying
```

## Use Cases

### Pre-Ticket Research

Before working a ticket, look up the company:
```
/find-company "Acme"
```
This shows available documentation and resource counts.

### Verify PSA Sync

Check if company is synced with PSA:
```
/find-company "Acme"
```
The PSA ID field indicates sync status.

### Find Related Companies

Identify parent/child relationships:
```
/find-company "Acme Holdings"
```
Shows all divisions under the parent.

### Audit Archived Clients

Review historical client data:
```
/find-company "Old" --status archived
```

## Related Commands

- `/lookup-asset` - Find assets for a company
- `/search-articles` - Search company articles
- `/get-password` - Get company credentials
