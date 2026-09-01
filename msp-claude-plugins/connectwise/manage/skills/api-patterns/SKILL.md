---
name: "ConnectWise Manage API Patterns"
description: >
  ConnectWise PSA API fundamentals as they reach the cw_* tool surface:
  the conditions query syntax, page/pageSize pagination, rate limiting
  (60/min), and error-response handling.
when_to_use: >-
  When composing a conditions query, paginating with page/pageSize, reading an error
  response, or reasoning about the 60/min rate limit. Use when: connectwise api, api
  conditions, query builder connectwise, connectwise pagination, api rate limit,
  connectwise rest, or api error connectwise. Not for credentials: the MCP server
  authenticates, and no client in this plugin constructs a ConnectWise credential.
---

# ConnectWise PSA API Patterns

## Overview

The ConnectWise PSA REST API provides access to all PSA entities including tickets, companies, contacts, projects, and time entries. This skill covers authentication, query syntax, pagination, rate limiting, and best practices for API integration.

## Anti-triggers

"ConnectWise" is an umbrella brand over three products with three
unrelated APIs. Loading the wrong one produces auth failures that read
like permission problems:

- **ConnectWise Automate** — on-premise RMM server, `/cwa/api/v1/` base
  path, Bearer-token auth, and singular `condition=` filters. PSA
  public/private keys will not authenticate against it. Use
  `connectwise-automate-api-patterns`.
- **ConnectWise CPQ (Sell/Quosal)** — its own host and credential set
  again; use `connectwise-cpq-api-patterns`.

## Base URLs

| Region | Base URL |
|--------|----------|
| North America | `https://api-na.myconnectwise.net/{codebase}/apis/3.0/` |
| Europe | `https://api-eu.myconnectwise.net/{codebase}/apis/3.0/` |
| Australia | `https://api-au.myconnectwise.net/{codebase}/apis/3.0/` |

Replace `{codebase}` with your company identifier (e.g., `v4_6_release` or custom).

### Legacy URLs

Some instances may use legacy URLs:
```
https://api-na.myconnectwise.net/v4_6_release/apis/3.0/
https://api-staging.connectwisedev.com/v4_6_release/apis/3.0/
```

## Authentication

**Nothing in this plugin authenticates to ConnectWise PSA.** The
`connectwise-manage-mcp` server holds one credential set and performs every
call; skills and commands reach the API only through its `cw_*` tools.

That is why this skill documents *query syntax and response semantics* and not
credential construction: the header a request carries is the server's concern,
and a client that built one would be bypassing the boundary rather than using
it.

The parts of this skill that remain useful are the ones that describe **what to
send a tool and how to read what comes back** — the conditions grammar,
pagination, ordering, and error semantics below.

## Conditions Query Syntax

### Basic Syntax

```
conditions=field operator value
```

### Supported Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Equals | `status/id=1` |
| `!=` | Not equals | `status/id!=5` |
| `<` | Less than | `priority/id<3` |
| `<=` | Less than or equal | `priority/id<=2` |
| `>` | Greater than | `dateEntered>2024-01-01` |
| `>=` | Greater than or equal | `dateEntered>=2024-01-01` |
| `contains` | Contains substring | `summary contains "email"` |
| `like` | Pattern match | `summary like "%email%"` |
| `in` | In list | `status/id in (1,2,3)` |
| `not in` | Not in list | `status/id not in (5)` |

### Field References

Use `/` to reference nested fields:

```
company/id=12345
status/name="New"
contact/firstName contains "John"
```

### Combining Conditions

**AND (default):**
```
conditions=company/id=12345 and status/id!=5 and priority/id<=2
```

**OR:**
```
conditions=status/id=1 or status/id=2
```

**Complex:**
```
conditions=(status/id=1 or status/id=2) and company/id=12345
```

### Date Conditions

**Date format:** `YYYY-MM-DD` or ISO 8601

```
conditions=dateEntered>=[2024-01-01]
conditions=dateEntered>=[2024-01-01T00:00:00Z] and dateEntered<[2024-02-01T00:00:00Z]
```

### String Conditions

**Exact match:**
```
conditions=summary="Email not working"
```

**Contains:**
```
conditions=summary contains "email"
```

**Like (wildcards):**
```
conditions=summary like "%email%"
conditions=company/identifier like "AC%"
```

### Null Checks

```
conditions=contact=null
conditions=assignedResource!=null
```

### URL Encoding

Special characters must be URL-encoded:

| Character | Encoded |
|-----------|---------|
| Space | `%20` |
| `=` | `%3D` |
| `<` | `%3C` |
| `>` | `%3E` |
| `"` | `%22` |

**Example** — the encoding a raw URL needs, shown so an error echoing an
encoded string is legible. A `conditions` argument passed to a `cw_*` tool is
**not** URL-encoded; send it as plain text:

```
company/id%3D12345%20and%20status/id!%3D5     <- as it appears in a URL
company/id=12345 and status/id!=5             <- as passed to conditions
```

## Pagination

### Request Parameters

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `page` | int | 1 | - | Page number (1-based) |
| `pageSize` | int | 25 | 1000 | Records per page |

### Example Request

```
cw_search_tickets
  conditions: "closedFlag=false"
  page:       1
  pageSize:   100
```

### Response Headers

The API sets these on a REST response. **A caller here never sees them:** the
MCP server parses the response body and returns that, so no header reaches the
tool result. They are documented as API knowledge, not as something to read.

| Header | Description |
|--------|-------------|
| `Link` | Contains next/prev page URLs |
| `X-Total-Count` | Total record count (if requested) |

`page` and `pageSize` are arguments on every `cw_search_*` and `cw_list_*`
tool. Paginate by incrementing `page` until a page returns fewer records than
`pageSize`. See [references/examples.md](references/examples.md) for worked
examples against the tool surface.

### Getting Total Count

**There is no count tool, and the header that would carry a total does not
reach the caller.** Neither of the two API-level approaches — reading
`X-Total-Count`, or a `/count` endpoint — is available through this surface.

To establish a total, page through the result with `cw_search_*` and count what
returns, stopping when a page yields fewer records than `pageSize`. On a wide
set that costs real requests against a 60/minute budget, so prefer narrowing
`conditions` to answering "how many" exactly.

## Rate Limiting

### Limits

| Limit | Value |
|-------|-------|
| Requests per minute | 60 |
| Per API member | Yes |

### Rate Limit Headers

The API sets these, and **a caller here cannot read them** for the same reason
as the response headers above. Budget by counting your own calls rather than by
monitoring a remaining count you have no access to.

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests per minute |
| `X-RateLimit-Remaining` | Requests remaining in window |
| `X-RateLimit-Reset` | Seconds until limit resets |

### 429 Response

When rate limited, you receive HTTP 429:

```json
{
  "code": "RateLimitExceeded",
  "message": "Rate limit exceeded. Try again in 30 seconds."
}
```

**The MCP server does not retry.** It issues one request and surfaces any
non-2xx response — 429 included — as a failed tool call, so backing off is the
caller's job. See [references/examples.md](references/examples.md) for what a
caller should do instead.

### Best Practices for Rate Limits

1. **Back off after a 429, and narrow the query** - the server will not do it for you
2. **Count your own calls** - the remaining-requests header is not visible from here, and the 60/minute budget is shared with every other caller using the same API member
3. **Batch operations** - Reduce total requests
4. **Avoid polling loops** - a repeated wide sweep is the usual way this limit gets hit

## Error Handling

Errors return the relevant HTTP status code plus a JSON body with `code`
and `message` fields, and per-field detail in `errors[]`. See
[references/errors.md](references/errors.md) for the complete HTTP status
code table, error response format, and common error codes.

## Common API Patterns

### Ordering

`orderBy` is an argument on every `cw_search_*` and `cw_list_*` tool:

```
cw_search_tickets
  conditions: "closedFlag=false"
  orderBy:    "priority/id asc, dateEntered desc"
```

### Field selection, child collections and custom fields

The API supports `fields`, `childconditions` and `customFieldConditions` as
query parameters. **None is exposed as a tool argument**, so from here a search
returns the server's field set and cannot be filtered to a projection or
constrained on a child collection. The parameters are recorded because they
explain what the API can do, not because they can be sent:

| Parameter | Effect | Available here |
|-----------|--------|----------------|
| `fields` | Return a projection only | No |
| `childconditions` | Filter on a child collection, e.g. `notes/text contains "update"` | No |
| `customFieldConditions` | Filter on a custom field | No |

Filter with `conditions` and read the fields you need from the returned record.

## Webhook Configuration

ConnectWise can POST entity-change events to a registered callback URL.
See [references/webhooks.md](references/webhooks.md) for the callback
payload shape and the registration request.

## Best Practices

1. **Store credentials securely** - Never commit to source control
2. **Handle errors gracefully** - Retry transient failures
3. **Use pagination** - Don't fetch unbounded results
4. **Select needed fields** - Reduce payload size
5. **Log API calls** - For debugging and audit
6. **Monitor usage** - Track API call patterns

## API Documentation

- [ConnectWise Developer Portal](https://developer.connectwise.com/)
- [REST API Reference](https://developer.connectwise.com/Products/ConnectWise_PSA/REST)
- [API Schema Browser](https://developer.connectwise.com/Products/ConnectWise_PSA/REST#swagger)

## Related Skills

- [ConnectWise Tickets](../tickets/SKILL.md) - Ticket management
- [ConnectWise Companies](../companies/SKILL.md) - Company management
- [ConnectWise Contacts](../contacts/SKILL.md) - Contact management
- [ConnectWise Projects](../projects/SKILL.md) - Project management
- [ConnectWise Time Entries](../time-entries/SKILL.md) - Time tracking
