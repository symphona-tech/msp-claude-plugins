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

**The server does not retry.** The pinned `connectwise-manage-mcp` client issues
a single `fetch` and throws on any non-2xx response, 429 included — there is no
backoff, no `Retry-After` handling, and no retry loop anywhere in it. A rate-limit
error therefore surfaces to the caller as a failed tool call.

So retry is the **caller's** responsibility, and the caller is an agent rather
than a request loop:

- On a rate-limit failure, **wait before retrying** — the vendor's message names
  a delay, and the documented limit is 60 requests per minute against the API
  member the server authenticates as. That budget is shared by every concurrent
  caller, so a wide sweep by one agent throttles everyone else mid-task.
- **Narrow the query rather than repeating it.** A retry of the same wide sweep
  reproduces the condition that caused the failure.
- On a permission or validation error, **stop**. Retrying will not change it.
- **Never fall back to another route to ConnectWise — there is none.**

Prefer avoiding the limit to recovering from it: a narrower `conditions` and a
smaller `pageSize` cost less than a backoff.

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
