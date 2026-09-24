# Hudu Articles Tool Reference

Every operation is an MCP tool call to the `hudu-mcp` server. The plugin builds no HTTP requests and holds no Hudu credential.

| Tool | What it does |
|------|--------------|
| `hudu_list_articles` | Lists one page of articles |
| `hudu_get_article` | Reads one article |
| `hudu_create_article` | Creates an article |
| `hudu_update_article` | Updates the fields given on an article |
| `hudu_archive_article` | Archives an article (one-way through this plugin) |
| `hudu_delete_article` | Deletes an article permanently |
| `hudu_list_folders` | Lists one page of folders |

## List Articles

`hudu_list_articles` takes optional `company_id`, `name`, `draft`, `page` and `page_size`.

**By Company:**
```json
{ "company_id": 123 }
```

**By Name:**
```json
{ "name": "backup" }
```

**With Pagination:**
```json
{ "company_id": 123, "page": 2, "page_size": 25 }
```

The tool returns one page: default `page_size` 25, `page` is 1-indexed, and there is no total count — "Found N articles" counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none. Each record is the full article, including its HTML `content` and `created_at`/`updated_at`.

`name` is the only text filter. No tool searches article `content`; matching words in the body means paging through articles and reading `content`, which is expensive.

## Get Single Article

`hudu_get_article` takes required `id`.

```json
{ "id": 456 }
```

**Returned record (abridged):**
```json
{
  "id": 456,
  "name": "Backup Procedure - Daily Operations",
  "content": "<h1>Backup Procedure</h1><h2>Overview</h2><p>The daily backup runs at 10PM...</p>",
  "company_id": 123,
  "company_name": "Acme Corporation",
  "folder_id": 15,
  "folder_name": "Procedures",
  "draft": false,
  "slug": "backup-procedure-daily-operations",
  "created_at": "2024-06-15T10:30:00.000Z",
  "updated_at": "2025-12-01T14:22:00.000Z",
  "url": "https://your-company.huducloud.com/a/backup-procedure-daily-operations-abcdef"
}
```

## Create Article

`hudu_create_article` takes required `name` and optional `content` (HTML), `company_id`, `folder_id`, `draft` and `enable_sharing`. Omitting `company_id` creates a global article visible to every company.

**Company-specific article:**
```json
{
  "name": "New User Setup Procedure",
  "company_id": 123,
  "folder_id": 20,
  "content": "<h1>New User Setup Procedure</h1><h2>Overview</h2><p>This procedure covers setting up a new user account for Acme Corporation.</p><h2>Prerequisites</h2><ul><li>Active Directory access</li><li>Microsoft 365 admin access</li></ul><h2>Steps</h2><ol><li>Create AD account with naming convention: first.last</li><li>Assign Microsoft 365 E3 license</li><li>Configure email signature using company template</li><li>Add to appropriate security groups</li></ol>"
}
```

**Global article (no company_id):**
```json
{
  "name": "Standard Password Policy",
  "content": "<h1>Standard Password Policy</h1><p>All managed client accounts must follow these requirements...</p><ul><li>Minimum 14 characters</li><li>Must include uppercase, lowercase, numbers, and symbols</li><li>Rotate every 90 days</li></ul>"
}
```

No tool uploads a file, image or attachment to an article; those are added in the Hudu web UI.

## Update Article

`hudu_update_article` takes required `id` and optional `name`, `content`, `company_id`, `folder_id`, `draft` and `enable_sharing`. Only the fields given are sent; `content` replaces the whole HTML body, so read the article with `hudu_get_article` first when editing part of it.

```json
{
  "id": 456,
  "content": "<h1>Backup Procedure - Updated</h1><p>Updated backup schedule...</p>",
  "draft": false
}
```

## Delete Article

`hudu_delete_article` takes required `id`. Deletion is irreversible, and it is a per-API-key permission in Hudu; a disabled delete permission surfaces as a tool error.

## Archive Article

`hudu_archive_article` takes required `id`. Its title says "(reversible)", but no tool unarchives an article: restoring one is a manual action in the Hudu web UI. Treat archiving as one-way through this plugin.

## Folder Management

### List Folders

`hudu_list_folders` takes optional `company_id`, `name`, `page` and `page_size`, and returns one page with no total count, like every list tool.

**By Company:**
```json
{ "company_id": 123 }
```

### Creating or Nesting Folders

No tool creates, renames, nests or moves a folder. Folders, including nested ones, are created in the Hudu web UI; the tools place an article into an existing folder by passing its `folder_id` to `hudu_create_article` or `hudu_update_article`.
