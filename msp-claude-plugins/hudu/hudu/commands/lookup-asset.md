---
description: Find an asset in Hudu by name, hostname, serial number, or IP address
argument-hint: "<query> [company] [layout]"
arguments: [query, company, layout]
---

# Lookup Hudu Asset

Find an asset in Hudu by various identifiers.

The Hudu MCP server must be connected.

## Steps

1. **Parse search query**
   - Determine if query is name, hostname, serial number, or IP
   - Resolve company name to ID if provided
   - Map layout filter to asset layout ID

2. **Execute search**
   - A name uses the `name` filter and a serial number uses `primary_serial` on `hudu_list_assets`
   - A hostname or IP lives in layout-defined custom fields, which no tool filters on: page through `hudu_list_assets` (narrowed by `company_id` and `asset_layout_id` where given) and match each asset's `fields`
   - Every list call returns one page (default 25) with no total count; request `page` 1, 2, 3… until a page returns fewer items than `page_size` before reporting a result as complete
   - Use `hudu_get_asset` for full detail on a match

3. **Format and return results**
   - Display asset details with key information
   - Include quick actions for further operations

## Tools

In call order:

1. `hudu_list_companies` with `name` — resolve the company filter to a `company_id`
2. `hudu_list_asset_layouts` with `name` — resolve the layout filter to an `asset_layout_id`
3. `hudu_list_assets` with `company_id`, `asset_layout_id`, and `name` or `primary_serial`, plus `page` — find candidates, paging until a short page
4. `hudu_get_asset` with `id` — full detail for a single match

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| query | string | Yes | - | Search term (name, hostname, serial, IP) |
| company | string | No | - | Company name filter |
| layout | string | No | - | Asset layout name filter (server/workstation/network) |

## Examples

### Search by Name

```
/lookup-asset "DC-01"
```

### Search by Hostname

```
/lookup-asset "dc-01.acme.local"
```

### Search by Serial Number

```
/lookup-asset "ABC123456789"
```

### Search by IP Address

```
/lookup-asset "192.168.1.10"
```

### Filter by Company

```
/lookup-asset "DC-01" --company "Acme Corp"
```

### Filter by Layout

```
/lookup-asset "firewall" --layout "Network Device"
```

### Combined Filters

```
/lookup-asset "01" --company "Acme" --layout "Server"
```

## Output

### Single Match

```
Found 1 asset matching "DC-01"

Asset: DC-01
------------------------------------------------------------
Company:       Acme Corporation
Layout:        Server
Serial:        ABC123456789
Model:         Dell PowerEdge R740

Custom Fields:
  Hostname:          dc-01.acme.local
  IP Address:        192.168.1.10
  Operating System:  Windows Server 2022
  RAM (GB):          32
  Warranty Expiry:   2027-01-15 (730 days remaining)

Last Updated:  2025-12-01

Quick Actions:
  - View in Hudu: [link]
  - Related passwords: /get-password --company "Acme Corp" "DC-01"
  - View articles: /search-articles "DC-01" --company "Acme Corp"
------------------------------------------------------------
```

### Multiple Matches

```
Found 3 assets matching "DC"

+------------------+------------------+----------+----------------+-------------+
| Name             | Company          | Layout   | Serial         | Updated     |
+------------------+------------------+----------+----------------+-------------+
| DC-01            | Acme Corp        | Server   | ABC123456789   | 2025-12-01  |
| DC-02            | Acme Corp        | Server   | DEF987654321   | 2025-11-15  |
| NYC-DC-01        | Acme East        | Server   | GHI456789012   | 2025-10-20  |
+------------------+------------------+----------+----------------+-------------+

Refine search:
  - Add company: /lookup-asset "DC" --company "Acme Corp"
  - Add layout: /lookup-asset "DC" --layout "Server"
```

### No Results

```
No assets found matching "XYZ-SERVER"

Suggestions:
  - Check spelling of the search term
  - Try a partial name match
  - Remove filters to broaden search
  - Search by IP or serial number instead

Example searches:
  /lookup-asset "XYZ"
  /lookup-asset "192.168"
```

## Filter Reference

### Common Layout Names

| Layout | Description |
|--------|-------------|
| Server | Physical or virtual servers |
| Workstation | End-user devices |
| Network Device | Routers, switches, firewalls |
| Printer | Print devices |
| Application | Software/services |
| Microsoft 365 | M365 tenant details |
| Backup | Backup configurations |

Note: Asset layout names are custom per Hudu instance. Use `hudu_list_asset_layouts` to see available layouts.

## Error Handling

### No Results

```
No assets found matching "invalid-search"

Suggestions:
  - Verify the search term is correct
  - Try searching by a different identifier
  - Check if the asset exists in Hudu
```

### Invalid Company

```
Company not found: "Acm"

Did you mean?
  - Acme Corporation
  - Acme East Division
  - Acme Industries

Try: /lookup-asset "DC-01" --company "Acme Corporation"
```

### Tool Error

```
Hudu tool error: Authentication failed - invalid API key

Every Hudu tool is failing, so the Hudu MCP server's API key is the
problem. Report it to whoever operates the Hudu MCP server; nothing in
this plugin can read or change the key.
```

### Rate Limited

```
Hudu tool error: Rate limit exceeded and max retries reached

The server already retried. Wait before retrying.
```

## Related Commands

- `/search-articles` - Search knowledge base articles
- `/get-password` - Get credentials for an asset
- `/find-company` - Find company details
