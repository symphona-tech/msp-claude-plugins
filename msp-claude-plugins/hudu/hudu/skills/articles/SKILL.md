---
name: "Hudu Articles"
description: >
  Hudu knowledge base articles: HTML content format, company-scoped vs global articles, article folders, drafts vs published, the hudu-mcp article and folder tools, and name-filtered search, templating, and documentation-health patterns.
when_to_use: >-
  When creating, searching, updating, or managing Hudu documentation articles and their folders.
  Use when: hudu article, hudu knowledge base, hudu kb, hudu documentation, hudu runbook, hudu
  procedure, knowledge base article, article management, or hudu docs.
---

# Hudu Articles Management

## Overview

Articles in Hudu serve as the knowledge base, providing a place for runbooks, procedures, network diagrams, SOPs, and general documentation. Articles support rich HTML content, can be organized into folders, and can be scoped to specific companies or kept as global (shared across all companies). MSP technicians rely on articles to quickly find procedures and reference documentation during troubleshooting.

## Anti-triggers

- **The same knowledge base in IT Glue** — IT Glue calls these Documents;
  use `itglue-documents`. Both platforms say "article", "document", and
  "runbook" interchangeably, so the vendor name is the only signal.
- **A structured record rather than prose** — if the thing has fields
  (make, model, IP, expiry) it belongs on an asset layout, not in article
  HTML; use `hudu-assets`.
- **A credential mentioned in a runbook** — passwords have their own
  endpoint and their own audit trail; use `hudu-passwords`. Do not let an
  agent paste a credential into article body HTML to "keep it together".
- **A ticket resolution note** — work notes belong on the ticket in the
  PSA, not in the knowledge base; use `autotask-ticket-notes-attachments`
  or `connectwise-psa-tickets`.

## Key Concepts

### Article Scope

Articles can be scoped in two ways:

| Scope | Description | Use Case |
|-------|-------------|---------|
| Company-specific | Tied to a single company | Network diagram for Acme Corp |
| Global | Available across all companies | Standard new user setup procedure |

Scope is controlled entirely by `company_id` — omit it (or set it to null) to create a global article.

### Article Folders

Folders organize articles within a company or globally, and can be nested. `hudu_list_folders` lists them; no tool creates, renames or moves a folder, so the hierarchy is built in the Hudu web UI and an article is placed into an existing folder by its `folder_id`:

```
Company: Acme Corporation
+-- Articles
    +-- Procedures
    |   +-- Backup Procedure
    |   +-- Disaster Recovery Plan
    +-- Network
    |   +-- Network Overview
    |   +-- IP Addressing Scheme
    +-- Onboarding
        +-- New User Setup
        +-- Hardware Deployment
```

### Article Content

Article content is stored as HTML. Hudu's editor supports:

- Headings, paragraphs, lists
- Tables
- Images (inline and uploaded — no tool uploads an image or file attachment; uploads are made in the Hudu web UI)
- Code blocks
- Embedded passwords (referenced by ID)
- Links to other Hudu resources

### Draft vs Published

Articles can be saved as drafts before publishing:

| State | Description |
|-------|-------------|
| Draft | Work in progress, not visible to all users |
| Published | Visible to users with appropriate permissions |

### Fields

Key fields: `id`, `company_id`, `name` (required), `content` (HTML), `folder_id`, `draft`, `slug`, `created_at`, `updated_at`, `url`.

See [references/fields.md](references/fields.md) for the complete field reference.

## Tools

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

| Tool | What it does |
|------|--------------|
| `hudu_list_articles` | Lists one page of articles, filtered by `company_id`, `name` and `draft` |
| `hudu_get_article` | Reads one article by `id` |
| `hudu_create_article` | Creates an article |
| `hudu_update_article` | Updates the fields given on an existing article |
| `hudu_archive_article` | Archives an article — one-way through this plugin |
| `hudu_delete_article` | Deletes an article permanently |
| `hudu_list_folders` | Lists one page of folders, filtered by `company_id` and `name` |

`hudu_list_articles` takes optional `company_id`, `name`, `draft`, `page` and `page_size`. It returns one page (default `page_size` 25, `page` 1-indexed) of full records, each including its HTML `content`, and carries no total count: "Found N" counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none.

`hudu_create_article` takes required `name` and optional `content` (HTML), `company_id`, `folder_id`, `draft` and `enable_sharing`. `hudu_update_article` takes required `id` and the same optional fields, and sends only the fields given.

`hudu_archive_article` and `hudu_delete_article` each take required `id`. `hudu_list_folders` takes optional `company_id`, `name`, `page` and `page_size`.

No tool creates, renames or moves a folder, unarchives an article, or uploads a file or image to an article.

See [references/api.md](references/api.md) for the complete tool reference with argument and record examples.

## Common Workflows

### Create Comprehensive Runbook

1. Resolve the target folder: call `hudu_list_folders` with the `company_id` (and `name` if known) and take the matching folder's `id`. If no folder fits, create the article at root level and tell the user the folder must be created in the Hudu web UI — no tool creates one.
2. Build the HTML `content`: an `<h1>` title, an `<h2>Overview</h2>` paragraph, an `<h2>Prerequisites</h2>` `<ul>` list and an `<h2>Procedure</h2>` `<ol>` list. Escape any user-supplied text before placing it in the HTML.
3. Call `hudu_create_article` with `name`, `company_id`, `folder_id` (if resolved) and `content`.

```json
{
  "name": "New User Setup Procedure",
  "company_id": 123,
  "folder_id": 20,
  "content": "<h1>New User Setup Procedure</h1><h2>Overview</h2><p>...</p><h2>Procedure</h2><ol><li>...</li></ol>"
}
```

### Article Search

No tool does full-text search. `hudu_list_articles` matches on `name` only (plus the `company_id` and `draft` filters), so a word that appears only in an article's body is not found by the `name` filter.

1. Call `hudu_list_articles` with `name` set to the query (and `company_id` if scoped) for title matches.
2. For body matches, page through `hudu_list_articles` without `name` — `page` 1, 2, 3… until a page returns fewer items than `page_size` — and match the query against each record's HTML `content`. This reads every article in scope and is expensive; scope it by `company_id` wherever possible and tell the user when the scan was limited.

### Documentation Health Check

1. Page through `hudu_list_articles` with the `company_id` until a page returns fewer items than `page_size`. Each record already carries `draft`, `content` and `updated_at`, so no per-article `hudu_get_article` call is needed.
2. From the collected records, count the total, the drafts (`draft: true`), those with `updated_at` in the last 30 days, list those with `updated_at` older than a year as stale, and list those whose `content` is missing or under about 50 characters as empty.

### Clone Article to Another Company

Folder IDs are company-scoped, so do not carry `folder_id` across companies — resolve a folder in the target company first with `hudu_list_folders`; if none fits, create the copy at root level.

1. Call `hudu_get_article` with the source `id`.
2. Call `hudu_create_article` with `name` (the new name, or the source name), `company_id` set to the target company, the source `content`, and the target `folder_id` if one was resolved.

## Article Templates

Standard HTML skeletons for Network Overview and Disaster Recovery Plan articles are in
[references/templates.md](references/templates.md).

## Gotchas

- **Folder IDs are scoped to a company.** Passing a `folder_id` belonging to another company makes the tool return `Validation error`; on failure, retry without `folder_id` to create at root level.
- **A missing `company_id` is not an error** — it silently creates a global article visible to every company. Double-check before creating client-specific content.
- **Content is raw HTML.** Values interpolated into `content` are not escaped by the API; sanitize any untrusted input before writing.
- **Archive is a distinct tool** (`hudu_archive_article`), not an `archived` field on `hudu_update_article`. It is one-way through this plugin: the tool's title says "(reversible)", but no tool unarchives an article, so restoring one is a manual action in the Hudu web UI. Do not archive expecting to undo it.
- **Delete is irreversible.** `hudu_delete_article` removes the article permanently; deletion is a per-API-key permission in Hudu, and a disabled one surfaces as a tool error.
- **List results are one page.** Every `hudu_list_articles` call returns at most `page_size` records and no total; never report the first page as the whole knowledge base.

See [references/errors.md](references/errors.md) for the complete error and validation table plus a recovery pattern.

## Best Practices

1. **Use consistent structure** - Follow templates for standard articles
2. **Organize with folders** - Keep a logical folder hierarchy per company (folders are created in the Hudu web UI; the tools only place articles into existing ones)
3. **Use global articles** - Share standard procedures across all companies
4. **Include visual aids** - Add diagrams, screenshots, and tables (images are uploaded in the Hudu web UI; no tool uploads them)
5. **Include metadata** - Add last reviewed date and author at the top
6. **Link related resources** - Reference assets and passwords

## Related Skills

- [Hudu Companies](../companies/SKILL.md) - Article company scope
- [Hudu Assets](../assets/SKILL.md) - Related asset references
- [Hudu Passwords](../passwords/SKILL.md) - Embedded credentials
