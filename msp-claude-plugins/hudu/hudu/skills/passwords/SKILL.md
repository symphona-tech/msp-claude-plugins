---
name: "Hudu Passwords"
description: >
  Hudu secure credential storage through the hudu_*_asset_password MCP tools (the UI calls these "Passwords"), company scoping and password folders, TOTP secrets, per-API-key password permissions, rotation workflows, and output-safety rules for handling plaintext credential values.
when_to_use: >-
  When storing, retrieving, rotating, or auditing credentials in Hudu, or when a password tool returns "Authentication failed - invalid API key" while other Hudu tools work. Use when: hudu password, hudu credential, credential
  lookup, password management, secure credentials, hudu credentials, password storage,
  credential documentation, password access, or asset password.
---

# Hudu Passwords Management

## Overview

Passwords in Hudu (called "asset passwords" in the API) provide secure credential storage scoped to companies. They allow MSP technicians to store, organize, and retrieve credentials for client infrastructure, applications, and services. Password access can be restricted at the API key level, and all access is logged in Hudu's activity logs.

**Critical naming note:** The Hudu UI calls these "Passwords," but the tools call them asset passwords: `hudu_list_asset_passwords`, `hudu_get_asset_password`, `hudu_create_asset_password`, `hudu_update_asset_password` and `hudu_delete_asset_password`.

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

## Anti-triggers

- **A credential needed to authenticate a tool call** — Hudu passwords document *the customer's* credentials. They are never the connector's own auth: the MCP server holds the Hudu API key, and nothing in this plugin can read it. An agent that reaches here to "find the API key" has taken a wrong turn.
- **A credential stored on an asset rather than as a password record** —
  many MSPs put licence keys and service accounts in asset custom fields.
  Those are not `asset_passwords`; use `hudu-assets`.
- **The same credential in IT Glue** — the other documentation platform in
  this marketplace stores passwords too, with its own permission model.
  Start from `itglue-api-patterns`.
- **Resetting or rotating the credential on the actual system** — this
  skill updates the documented value only. Changing the real password is a
  tenant or directory operation; use `cipp-users` or `m365-users`. Editing
  the record without changing the system leaves documentation that is
  confidently wrong.

## Key Concepts

### Password Organization

Passwords are organized by:

- **Company** - Each password belongs to a specific company
- **Password Folders** - Hierarchical folder structure within a company
- **Name** - Descriptive name identifying the credential

```
Company: Acme Corporation
+-- Passwords
    +-- Infrastructure
    |   +-- Domain Admin - ACME
    |   +-- Local Admin - Servers
    |   +-- vCenter Admin
    +-- Network
    |   +-- Firewall Admin
    |   +-- Switch Admin
    |   +-- WiFi Controller
    +-- Applications
    |   +-- ERP Admin
    |   +-- CRM Admin
    +-- Cloud Services
        +-- Microsoft 365 Global Admin
        +-- AWS Root Account
```

### API Key Password Permission

API keys in Hudu can be configured to allow or deny password access:

| Permission | Effect |
|------------|--------|
| Enabled | API key can read/write password values |
| Disabled | Every password tool returns `Authentication failed - invalid API key` |

This is configured per API key in Admin > API Keys. When the password tools fail with that message while every other Hudu tool works, the key is valid and password access is disabled on it: Hudu answers 401 `Bad credentials` for a valid key lacking password permission, and the server maps every 401 to that message. Report that credential access is not enabled for this connection; do not report the key as expired or ask for it to be rotated.

### Security Audit Trail

Hudu records password access in its activity log — who accessed it, when, and what action (view, create, update, delete). An operator reviews that log in the Hudu web UI; no tool in this plugin reads it, so an agent cannot produce or verify a password-access audit trail.

### Fields

Core fields: `company_id` (required), `name` (required), `username`, `password`, `url`, `description`, `password_type`, `otp_secret`, `password_folder_id`.

See [references/fields.md](references/fields.md) for the complete field reference.

## Tools

| Tool | What it does |
|------|--------------|
| `hudu_list_asset_passwords` | List password records, filtered by company, name or search term (one page per call) |
| `hudu_get_asset_password` | Read one password record by id |
| `hudu_create_asset_password` | Create a password record |
| `hudu_update_asset_password` | Change fields on an existing password record |
| `hudu_delete_asset_password` | Delete a password record (irreversible) |

`hudu_list_asset_passwords` takes optional `company_id`, `name`, `search`, `page` and `page_size`. It returns one page (default `page_size` 25, `page` 1-indexed) with no total count; "Found N …" counts that page only. Request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none.

`hudu_get_asset_password` takes `id`.

`hudu_create_asset_password` takes `company_id` and `name`, and optional `username`, `password`, `url`, `description`, `password_type`, `otp_secret`, `password_folder_id`, `passwordable_type`, `passwordable_id` and `in_portal`.

`hudu_update_asset_password` takes `id`, and optional `name`, `username`, `password`, `url`, `description`, `password_type` and `otp_secret`. It sends only the fields given. It cannot change `company_id`, `password_folder_id`, `passwordable_*` or `in_portal`; only create sets them.

`hudu_delete_asset_password` takes `id`. Deletion is irreversible, and it fails if deletion is disabled on the server's API key.

**Both read tools return the plaintext `password` value.** `hudu_list_asset_passwords` returns full records, `password` and `otp_secret` included, so list output is as sensitive as get output. Treat every result from these tools as sensitive.

No tool lists, creates or moves password folders, and no tool reads Hudu's activity log.

See [references/api.md](references/api.md) for the complete tool reference with arguments and an abridged returned record.

## Output Safety

**Never include actual password values in:**
- Correlation summaries or reports
- Log files
- Chat output or conversation history
- Error messages
- Any output that may be visible to unauthorized users

When displaying password information, always mask the actual value:

```
Password: Domain Admin - ACME
Username: administrator@acme.local
Password: **************
URL:      https://dc01.acme.local
```

## Common Workflows

### Secure Password Creation

1. Resolve the company with `hudu_list_companies` (`name`) to get its `id`.
2. Call `hudu_create_asset_password` with `company_id`, `name`, `username`, `password`, `url`, `password_type`, and a `description` that starts `Created: <date>` followed by `Purpose: <purpose>`. Pass `password_folder_id` only if you already know the folder id (for example from an existing record's `password_folder_id`); no tool lists password folders, and update cannot move a record into a folder later.
3. Confirm the result by name and id only; do not echo the password value.

### Password Rotation Workflow

Hudu keeps no rotation history of its own — append rotation dates to `description` so the audit trail survives.

1. Call `hudu_get_asset_password` with `id` to read the current `description` (use it for the note only; never repeat the old value).
2. Call `hudu_update_asset_password` with `id`, the new `password`, and `description` set to the existing description plus a new line `Rotated: <date> - <reason>`. Passing `description` replaces it, so include the existing text.
3. Report the rotation by record name and date only.

### Password Search by Context

`hudu_list_asset_passwords` filters on `company_id`, `name` and `search`. To match a server name against description or URL reliably, read the records yourself:

1. Call `hudu_list_asset_passwords` with `company_id` (and `search` set to the server name, if you want Hudu to narrow the page first).
2. Page through `page` 1, 2, 3… until a page returns fewer items than `page_size` or none.
3. Keep records whose `name`, `description` or `url` contains the server name, case-insensitively. Report name, username and URL; never the password.

### Find Stale Passwords

1. Call `hudu_list_asset_passwords` with `company_id`, paging until a page returns fewer items than `page_size` or none.
2. Keep records whose `updated_at` is older than the cutoff (90 days by default).
3. Report `id`, `name`, `username`, `updated_at` and days since update for each. Never include the password value.

### Password Inventory Report

**NEVER include actual password values (or `otp_secret`) in reports.** The list tool returns them; drop them before building any output.

1. Call `hudu_list_asset_passwords` with `company_id`, paging until a page returns fewer items than `page_size` or none. A report built from the first page alone is incomplete.
2. Group records by `password_type`, using `Uncategorized` where it is empty.
3. For each record report `name`, `username`, `url` and `updated_at` only.

## Gotchas

- **The tools say `asset_password`, not `password`.** The UI name and the tool names differ.
- **`Authentication failed - invalid API key` on a password tool is a key-permission problem, not a bad key.** Password access is a per-API-key toggle in Admin > API Keys. Hudu answers 401 `Bad credentials` for a valid key lacking password permission, and the server reports every 401 as that message. If every other Hudu tool works, report that credential access is not enabled for this connection; do not call the key invalid or ask for it to be rotated. Only if *every* tool returns it is the key itself the problem.
- **List output contains plaintext passwords.** `hudu_list_asset_passwords` returns full records including `password` and `otp_secret`, exactly as `hudu_get_asset_password` does.
- **One page per call, no total count.** "Found N …" counts that page only; page until a page returns fewer items than `page_size` or none before calling a result complete.
- **`url` appears twice in responses** with different meanings: the credential's login URL on create/update, and the Hudu record URL in the read payload's metadata. Do not round-trip it blindly.
- **Every read is logged.** Bulk enumeration of passwords generates a visible audit trail in Hudu's activity log (reviewed in the Hudu web UI; no tool here reads it); scope by `company_id` rather than sweeping the tenant.
- **Deletion is unrecoverable and drops the audit context.** Prefer keeping stale credentials with a rotation note.

See [references/errors.md](references/errors.md) for the complete failure-mode table and secure error handling.

## Security Best Practices

### Access Control

1. **Restrict API key permissions** - Only enable password access on keys that need it. The key's permission toggles (password access, deletion) are the whole of what it can restrict; Hudu cannot limit a key to certain companies or IPs.
2. **Scope reads by `company_id`** - As practice, not as a control: the key can reach every company regardless.
3. **Regular access reviews** - Audit who has API keys with password access

### Password Hygiene

1. **Regular rotation** - Rotate passwords on schedule (90 days recommended)
2. **Unique passwords** - Never reuse passwords across systems
3. **Track changes** - Update description when passwords are rotated
4. **Monitor stale passwords** - Alert on passwords not updated recently

### Documentation Hygiene

1. **Use descriptive names** - Include system name and account type (e.g., "Domain Admin - ACME")
2. **Set password type** - Classify passwords (Administrative, Network, Application, etc.)
3. **Organize with folders** - Create a logical folder hierarchy per company (in the Hudu web UI; no tool creates password folders)
4. **Document purpose** - Use the description field to explain what the password is for
5. **Track URLs** - Always include the login URL when applicable
6. **Include 2FA** - Store TOTP secrets with the `otp_secret` field

## Related Skills

- [Hudu Companies](../companies/SKILL.md) - Password company scope
- [Hudu Assets](../assets/SKILL.md) - Device-related credentials
- [Hudu Articles](../articles/SKILL.md) - Embedding passwords in articles
