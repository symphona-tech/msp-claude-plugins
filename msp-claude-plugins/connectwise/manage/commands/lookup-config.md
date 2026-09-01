---
description: Search for configuration items (assets) in ConnectWise PSA
argument-hint: "[query] [company_id] [type] [status] [limit]"
arguments: [query, company_id, type, status, limit]
---

# Look Up ConnectWise PSA Configuration Items

Search for configuration items (assets) by name, serial number, tag number, or company.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold configuration read permissions

## Tools used

| Step | Tool |
|------|------|
| Resolve company | `cw_search_companies` |
| Search configurations | `cw_search_configurations` |
| Read one configuration | `cw_get_configuration` |

## Steps

1. **Resolve the company (if named)**
   - `cw_search_companies` with `conditions=name contains "<company>"`

2. **Build the conditions string**
   - Combine the caller's filters into ConnectWise conditions syntax
   - Type and status filter **by name**, not by resolved ID — see below

3. **Execute the search**

   ```
   cw_search_configurations
     conditions: "company/id=12345 and status/name=\"Active\""
     orderBy:    "name asc"
     pageSize:   <limit>
   ```

4. **Read one configuration in full (if a single result)**
   - `cw_get_configuration` with `id=<configuration_id>`

5. **Format and return results**

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| query | string | No* | - | Search term (name, serial, tag) |
| company_id | integer | No | - | Filter by company ID |
| type | string | No | - | Configuration type filter |
| status | string | No | - | Configuration status filter |
| limit | integer | No | 25 | Max results (1-100) |

*Required if company_id not provided

## Examples

### Search by Name

```
/lookup-config "ACME-WS-001"
```

### Search by Serial Number

```
/lookup-config "SN123456789"
```

### Search by Tag Number

```
/lookup-config "TAG-001"
```

### Filter by Company

```
/lookup-config --company_id 12345
```

### Active Configurations Only

```
/lookup-config --company_id 12345 --status "Active"
```

### Combined Search

```
/lookup-config "Dell" --type "Workstation" --status "Active" --limit 50
```

### Company Servers

```
/lookup-config --company_id 12345 --type "Server" --status "Active"
```

## Output

### Search Results

```
Found 5 configuration items matching "ACME"

+--------+------------------+--------------+-----------+--------+------------------+
| ID     | Name             | Type         | Company   | Status | Serial Number    |
+--------+------------------+--------------+-----------+--------+------------------+
| 10001  | ACME-WS-001      | Workstation  | Acme Corp | Active | SN123456789     |
| 10002  | ACME-WS-002      | Workstation  | Acme Corp | Active | SN123456790     |
| 10003  | ACME-SRV-01      | Server       | Acme Corp | Active | SN987654321     |
| 10004  | ACME-SRV-02      | Server       | Acme Corp | Active | SN987654322     |
| 10005  | ACME-FW-01       | Firewall     | Acme Corp | Active | SN555555555     |
+--------+------------------+--------------+-----------+--------+------------------+

Quick Actions:
- View details: /lookup-config <name>
```

### Detailed View (Single Result)

```
================================================================================
Configuration Item: ACME-SRV-01
================================================================================

Basic Information:
  ID:             10003
  Name:           ACME-SRV-01
  Type:           Server - Windows
  Status:         Active

Company:
  Name:           Acme Corporation
  ID:             12345

Hardware Details:
  Manufacturer:   Dell
  Model:          PowerEdge R740
  Serial Number:  SN987654321
  Tag Number:     TAG-SRV-01

Network:
  IP Address:     192.168.1.10
  MAC Address:    00:1A:2B:3C:4D:5E
  Default Gateway: 192.168.1.1

Location:
  Site:           Main Office
  Address:        123 Main St, Anytown, USA

Warranty:
  Vendor:         Dell
  Expiration:     2027-06-15
  Type:           ProSupport Plus

Notes:
  Primary domain controller and file server.
  Critical system - 4 hour response SLA.

Installed Software:
  - Windows Server 2022 Datacenter
  - SQL Server 2019 Standard
  - Veeam Agent

Related Tickets (last 30 days):
  #54321 - Server slow response (Closed)
  #54100 - Disk space warning (Closed)

Created:        2024-01-15 by Admin
Last Updated:   2026-02-01

================================================================================
```

## Error Handling

### No Results

```
No configuration items found matching "XYZ123"

Suggestions:
- Check spelling
- Try partial name match
- Search by serial number or tag
- Broaden type/status filters

Example searches:
  /lookup-config "partial-name"
  /lookup-config --company_id 12345
```

### Missing Search Criteria

```
Error: Search criteria required

Please provide a query or company_id:
  /lookup-config "search term"
  /lookup-config --company_id 12345

You can also filter by type and status:
  /lookup-config --company_id 12345 --type "Server" --status "Active"
```

### Company Not Found

```
Error: Company ID 99999 not found

Use /search-company to find the correct company ID.
```

### Too Many Results

```
Found 247 configuration items (showing first 25)

Narrow your search:
- Add a more specific query
- Filter by type: --type "Server"
- Filter by status: --status "Active"
- Specify company: --company_id 12345

Use --limit to increase results (max 100).
```

### Rate Limited

```
Rate limited by ConnectWise API

Retrying in 5 seconds...
Successfully retrieved configurations.
```

### Permission Denied

```
Error: Permission denied

You do not have permission to view configuration items.
Contact your ConnectWise administrator.
```

## Not available through the tool surface

| Requested | Would need |
|-----------|------------|
| Validating a `--type` against the configured list | A configuration type tool |
| Validating a `--status` against the configured list | A configuration status tool |

Neither list can be enumerated, so a name the caller supplies **cannot be
checked before it is used**. Filter on it directly in the conditions string —
`type/name="Server"` — and report an empty result as *no match for that filter*
rather than as *no configurations exist*. The two readings are different, and
without the lists this command cannot tell them apart.

The reference tables below are **retained as domain knowledge, not as a
validated list**: they record what these values typically look like in a
ConnectWise instance, and any given instance may differ.

## Filter Reference

### Common Configuration Types

| Type | Description |
|------|-------------|
| Workstation | Desktop/laptop computers |
| Server | Physical servers |
| Server - Windows | Windows servers |
| Server - Linux | Linux servers |
| Virtual Machine | VM instances |
| Firewall | Network firewalls |
| Switch | Network switches |
| Router | Network routers |
| Printer | Printers/MFPs |
| Mobile Device | Phones/tablets |
| UPS | Backup power |
| Storage | NAS/SAN devices |

### Configuration Statuses

| Status | Description |
|--------|-------------|
| Active | Currently in use |
| Inactive | Not currently in use |
| Retired | Decommissioned |
| In Stock | Available for deployment |
| Pending | Awaiting setup |

## Related Commands

- `/get-ticket` - View ticket details (ticket-linked configurations are not retrievable)
- `/create-ticket` - Create ticket and link configuration
- `/check-agreement` - View agreement for configuration coverage
- `/search-tickets` - Find tickets by configuration
