---
description: Search for devices across NinjaOne organizations
argument-hint: "<query>"
arguments: [query]
---

Search for devices in NinjaOne matching the query "$ARGUMENTS.query".

## Arguments

- `query` (required) — Search query (hostname, IP, or organization name)

## Instructions

1. If the query looks like an organization name, first search organizations to get the org ID
2. Query devices with appropriate filters based on the search criteria
3. Present results in a table showing:
   - Device name/hostname
   - Organization
   - Device type (Workstation/Server)
   - Status (Online/Offline)
   - IP Address
   - Last contact time

## Tools

- `ninjaone_devices_list` — the primary call. Filter by `organization_id`, `device_class` or `online`, and page by `cursor` until no cursor comes back.
- `ninjaone_organizations_list` — resolve a client name to an `organization_id` first when the query names one.
- `ninjaone_devices_get` — full detail for one device once it has been located.

**Page to exhaustion.** `ninjaone_devices_list` returns one page at a time; reporting the first page as the whole result is how a search comes back plausibly short.

## Example Output

| Device | Organization | Type | Status | IP | Last Contact |
|--------|--------------|------|--------|-----|--------------|
| SERVER-01 | Acme Corp | Windows Server | Online | 192.168.1.10 | 2 min ago |
| WS-JOHN | Acme Corp | Workstation | Online | 192.168.1.25 | 5 min ago |

If no devices match, suggest alternative search terms or confirm the query.
