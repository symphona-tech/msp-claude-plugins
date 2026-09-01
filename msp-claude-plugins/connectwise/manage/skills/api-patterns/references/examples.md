# ConnectWise PSA tool examples

Worked examples against the `connectwise-manage-mcp` tool surface. Every call
goes through the server; nothing here constructs a credential or reaches the
vendor API directly.

## Searching with conditions

`conditions` takes ConnectWise query syntax verbatim, so most of the work is
composing the string rather than building a request.

```
cw_search_tickets
  conditions: "company/id=12345 and closedFlag=false"
  orderBy:    "priority/id asc, dateEntered desc"
  pageSize:   100
```

Quoting matters: values are double-quoted, dates are bracketed.

```
cw_search_tickets
  conditions: "status/name=\"In Progress\" and dateEntered>=[2026-02-01]"
```

## Pagination

`page` and `pageSize` are arguments on every `cw_search_*` and `cw_list_*`
tool. `pageSize` maxes at 1000 and defaults to 25.

To walk every page, increment `page` until a page returns fewer records than
`pageSize`:

```
cw_search_tickets  conditions: "closedFlag=false"  page: 1  pageSize: 250
cw_search_tickets  conditions: "closedFlag=false"  page: 2  pageSize: 250
cw_search_tickets  conditions: "closedFlag=false"  page: 3  pageSize: 250   -> 118 records, stop
```

**Prefer a narrower `conditions` over a full sweep.** The rate limit is 60
requests per minute against the API member the server authenticates as, and
that budget is shared by every concurrent caller — a wide sweep by one agent
will throttle everyone else's calls mid-task.

## Rate limiting and retries

Retry handling lives in the server, not in the caller. A `cw_*` tool that
returns an error has already exhausted whatever retry policy the server
applies, so **treat a failure as final** rather than looping on it.

What a caller should do instead:

- Narrow the query and try once more, if the failure was a timeout on a wide sweep
- Report the failure and stop, if it was a permission or validation error
- Never fall back to another route to ConnectWise — there is none

## Reading a write result

Write tools return the created or updated record. Read the identifier back
rather than assuming the values you sent:

```
cw_create_ticket  summary: "Email not working"  companyId: 12345  boardId: 1
  -> { "id": 54321, "summary": "Email not working", "status": {...}, ... }
```

The `status` a ticket lands in is the board's default, not necessarily what was
requested — check it before reporting the ticket as created in a given state.

## JSON Patch updates

`cw_update_ticket` and `cw_update_company` take JSON Patch operations. Group
related changes into one call so the update is atomic:

```
cw_update_ticket
  id: 54321
  operations: [
    {"op": "replace", "path": "status/id",   "value": 5},
    {"op": "replace", "path": "resolution",  "value": "DNS records corrected."}
  ]
```

`replace` overwrites. To append to a field, read it with `cw_get_ticket` first
and send the concatenated value.

## Related

- [errors.md](errors.md) — error response shapes
- [webhooks.md](webhooks.md) — callback configuration
- [SKILL.md](../SKILL.md) — the conditions grammar in full
