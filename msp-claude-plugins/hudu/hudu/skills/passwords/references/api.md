# Hudu Asset Passwords Tool Reference

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

## Tools

| Tool | What it does |
|------|--------------|
| `hudu_list_asset_passwords` | List password records, one page per call |
| `hudu_get_asset_password` | Read one password record by id |
| `hudu_create_asset_password` | Create a password record |
| `hudu_update_asset_password` | Change fields on an existing password record |
| `hudu_delete_asset_password` | Delete a password record (irreversible) |

## List Passwords

`hudu_list_asset_passwords` takes optional `company_id`, `name`, `search`, `page` and `page_size`.

```json
{ "company_id": 123, "name": "admin", "page": 1 }
```

It returns full records, including the plaintext `password` and `otp_secret`, so list output is as sensitive as get output.

### Paging

Each call returns one page: default `page_size` 25, `page` is 1-indexed, and the result carries no total count. The tool's "Found N …" message counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size`, or none. Reporting the first page as the whole result is the most plausible wrong answer here.

## Get Single Password

`hudu_get_asset_password` takes `id`.

Abridged returned record:

```json
{
  "id": 789,
  "company_id": 123,
  "company_name": "Acme Corporation",
  "name": "Domain Admin - ACME",
  "username": "administrator@acme.local",
  "password": "<plaintext value>",
  "url": "https://dc01.acme.local",
  "description": "Primary domain administrator account.",
  "password_type": "Administrative",
  "password_folder_id": 45,
  "password_folder_name": "Infrastructure",
  "otp_secret": null,
  "created_at": "2024-01-15T10:30:00.000Z",
  "updated_at": "2025-11-15T14:22:00.000Z"
}
```

The password value is returned in plaintext. See the output-safety rules in SKILL.md.

## Create Password

`hudu_create_asset_password` takes `company_id` and `name`, and optional `username`, `password`, `url`, `description`, `password_type`, `otp_secret`, `password_folder_id`, `passwordable_type`, `passwordable_id` and `in_portal`.

```json
{
  "company_id": 123,
  "name": "Domain Admin - ACME",
  "username": "administrator@acme.local",
  "password": "<value>",
  "url": "https://dc01.acme.local",
  "description": "Primary domain administrator account",
  "password_type": "Administrative",
  "password_folder_id": 45
}
```

No tool lists or creates password folders; take a `password_folder_id` from an existing record, or omit it.

## Update Password

`hudu_update_asset_password` takes `id`, and optional `name`, `username`, `password`, `url`, `description`, `password_type` and `otp_secret`. It sends only the fields given. It cannot change `company_id`, `password_folder_id`, `passwordable_type`, `passwordable_id` or `in_portal`.

```json
{
  "id": 789,
  "password": "<new value>",
  "description": "Password rotated on 2026-02-15. Previous rotation: 2025-11-15."
}
```

## Delete Password

`hudu_delete_asset_password` takes `id`. Deletion is irreversible, and fails if deletion is disabled on the server's API key. Prefer keeping passwords for audit purposes.

## Audit Logging

Hudu records password access in its activity log, which an operator reviews in the Hudu web UI. No tool in this plugin reads it.
