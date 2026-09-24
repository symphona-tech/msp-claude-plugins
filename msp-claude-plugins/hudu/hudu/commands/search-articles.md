---
description: Search Hudu knowledge base articles by keyword or phrase
argument-hint: "<query> [company] [limit]"
arguments: [query, company, limit]
---

# Search Hudu Articles

Search through Hudu knowledge base articles to find relevant documentation.

The Hudu MCP server must be connected.

## Steps

1. **Parse search parameters**
   - Extract search query terms
   - Resolve company name to ID if provided (`hudu_list_companies` with `name`)
   - Set result limit

2. **Execute search**
   - Title matches: call `hudu_list_articles` with `name` set to the query and `company_id` if resolved. Server-side matching is by article `name` only; no tool does full-text search.
   - Content matches: page through `hudu_list_articles` without `name` (`page` 1, 2, 3… until a page returns fewer items than `page_size`) and match the query against each record's HTML `content`, stripping tags first. This reads every article in scope and is expensive — prefer a company filter, and say in the output when the content scan was skipped or stopped early.
   - Each list call returns one page with no total count; "Found N" from the tool counts that page only, so never report the first page as the full result.

3. **Rank and format results**
   - Sort by relevance (name matches first)
   - Display with content snippets
   - Include quick actions

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| query | string | Yes | - | Search keywords or phrase |
| company | string | No | - | Company name filter |
| limit | int | No | 10 | Maximum results (1-50) |

## Examples

### Basic Search

```
/search-articles "backup procedure"
```

### Search in Company

```
/search-articles "disaster recovery" --company "Acme Corp"
```

### Limit Results

```
/search-articles "password" --limit 5
```

### Combined Filters

```
/search-articles "VPN setup" --company "Acme" --limit 20
```

## Output

### Results Found

```
Found 5 articles matching "backup procedure"

================================================================

1. Backup Procedure - Daily Operations
   Company: Acme Corporation
   Folder: Procedures > Backup
   Updated: 2025-12-10

   "...The daily backup procedure runs at 10PM and backs up all
   file server data to the NAS. Verify backup completion each
   morning by checking..."

   [View Article] [Open in Hudu]

================================================================

2. Backup Overview - Acme Corp
   Company: Acme Corporation
   Folder: Infrastructure
   Updated: 2025-11-25

   Backup Solution: Veeam Backup & Replication
   Backup Server: ACME-BKP-01
   Retention: 30 days local, 90 days cloud

   [View Article] [Open in Hudu]

================================================================

3. Disaster Recovery Plan
   Company: Acme Corporation
   Folder: Procedures > DR
   Updated: 2025-10-15

   "...In case of complete site failure, restore from backup
   using the disaster recovery procedure outlined below..."

   [View Article] [Open in Hudu]

================================================================

Showing 3 of 5 results. Use --limit 50 to see more.
```

### Single Detailed Result

When only one result is found, show full details:

```
Found 1 article matching "VPN setup Acme"

Article: VPN Setup Procedure
================================================================
Company: Acme Corporation
Folder: Procedures > Remote Access
Created: 2024-06-15
Updated: 2025-12-01

Content Preview:
================================================================

VPN Setup Procedure

Overview
This procedure covers setting up VPN access for remote users.

Prerequisites
- Active Directory account
- VPN client installed
- MFA token configured

Steps
1. Download the VPN client from...
2. Enter the server address: vpn.acme.com
3. Use your AD credentials...

================================================================

Related Resources:
- VPN Credentials: /get-password "VPN" --company "Acme Corp"
- Firewall: /lookup-asset "firewall" --company "Acme Corp"

[Open in Hudu]
```

### No Results

```
No articles found matching "nonexistent topic"

Suggestions:
  - Try different keywords
  - Use partial words (e.g., "back" instead of "backup")
  - Remove company filter to search all companies
  - Check global articles (shared across companies)

Example searches:
  /search-articles "backup"
  /search-articles "network" --company "Acme"
```

## Search Behavior

Hudu matches only the article `name` server-side. Content matching, multi-word AND and exact-phrase matching are done by the agent over the `content` of the articles it has paged through, so they are only as complete as that scan.

| Search Term | Matches |
|-------------|---------|
| Single word | Title (server-side `name` filter) and, if scanned, content containing word |
| Multiple words | All words must appear (AND), checked by the agent over fetched articles |
| "Quoted phrase" | Exact phrase match, checked by the agent over fetched articles |

## Error Handling

### No Results

```
No articles found matching "xyz123"

Try:
  - Broadening your search terms
  - Removing filters
  - Checking for typos
```

### Invalid Company

```
Company not found: "Acm"

Did you mean?
  - Acme Corporation
  - Acme East Division

Try: /search-articles "backup" --company "Acme Corporation"
```

### Too Many Results

```
Found 150+ articles matching "the"

Your search returned too many results. Please refine:
  - Add more specific keywords
  - Filter by company: --company "Acme"
  - Use quoted phrases: "backup procedure"
```

### Tool Error

```
Error searching Hudu articles: <tool error string>
```

Report the tool error string as returned. `Authentication failed - invalid API key` on every tool means the Hudu MCP server's key is not accepted; `Rate limit exceeded and max retries reached` means wait before retrying; `Server error: <status>` means the server already retried once. The plugin holds no Hudu credential and has no configuration to check.

## Tools

1. `hudu_list_companies` — resolve the company name to `company_id` (only when a company is given)
2. `hudu_list_articles` — `name` filter for title matches, then paged without `name` for content matches
3. `hudu_get_article` — read one result in full when a single article is shown in detail

## Related Commands

- `/lookup-asset` - Find assets
- `/get-password` - Get credentials
- `/find-company` - Find company details
