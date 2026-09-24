# Hudu Assets Tool Reference

Every operation is an MCP tool call. The plugin builds no HTTP requests and holds no Hudu credential.

## Tools

| Tool | What it does |
|------|--------------|
| `hudu_list_assets` | List assets, one page at a time |
| `hudu_get_asset` | Get one asset by id |
| `hudu_create_asset` | Create an asset |
| `hudu_update_asset` | Update an asset; only the fields given are sent |
| `hudu_archive_asset` | Archive an asset — one-way through this plugin |
| `hudu_delete_asset` | Delete an asset permanently |
| `hudu_list_asset_layouts` | List asset layouts |
| `hudu_get_asset_layout` | Get one asset layout with its field definitions |
| `hudu_create_asset_layout` | Create an asset layout |
| `hudu_update_asset_layout` | Update an asset layout |
| `hudu_list_relations` | List relations between records, one page at a time |

## List Assets

`hudu_list_assets` takes optional `company_id`, `asset_layout_id`, `name`, `primary_serial`, `archived`, `page` and `page_size`. Filters combine.

```json
{ "company_id": 123, "asset_layout_id": 5, "archived": false }
```

```json
{ "primary_serial": "ABC123456789" }
```

**Paging:** the tool returns one page — default `page_size` 25, `page` 1-indexed — and no total count. Its "Found N" message counts that page only. To enumerate, request `page` 1, 2, 3… until a page returns fewer items than `page_size` or none. Do not report the first page as the whole result.

Each asset comes back as a full record, including `fields` (the layout's custom field values), `company_name`, `archived`, `created_at` and `updated_at`. Custom fields cannot be filtered on; filter `fields` yourself after paging.

## Get Single Asset

`hudu_get_asset` takes `id` (required).

```json
{ "id": 789 }
```

## Create Asset

`hudu_create_asset` takes `company_id`, `asset_layout_id` and `name` (all required), and optional `custom_fields`, `primary_mail`, `primary_manufacturer`, `primary_model` and `primary_serial`.

`custom_fields` is an object with the layout's field labels as keys. Before writing it, call `hudu_get_asset_layout` for the layout and read `fields` to learn the labels, their `field_type`, and which are `required` — do not invent keys or guess the shape. A missing required field returns `Validation error`, which does not name the field.

**Server example** (the `custom_fields` keys stand for the labels the layout returned):
```json
{
  "company_id": 123,
  "asset_layout_id": 5,
  "name": "DC-01",
  "primary_serial": "ABC123456789",
  "primary_model": "Dell PowerEdge R740",
  "custom_fields": { "<Hostname label>": "dc-01.acme.local", "<IP Address label>": "192.168.1.10" }
}
```

## Update Asset

`hudu_update_asset` takes `id` (required) and optional `asset_layout_id`, `company_id`, `custom_fields`, `name`, `primary_mail`, `primary_manufacturer`, `primary_model` and `primary_serial`. Only the fields given are sent. `custom_fields` follows the same rule as on create: keys are the layout's field labels, read from `hudu_get_asset_layout`.

## Delete Asset

`hudu_delete_asset` takes `id` (required). Deletion is irreversible. Deletion is a per-API-key permission in Hudu; if it is disabled, the tool returns an error.

## Archive Asset

`hudu_archive_asset` takes `id` (required). Its tool title says "(reversible)", but that is wrong for this plugin: no tool unarchives an asset. Restoring an archived asset is a manual action in the Hudu web UI, so confirm before archiving. Archived assets are excluded unless `hudu_list_assets` is called with `archived: true`.

## List Asset Layouts

`hudu_list_asset_layouts` takes optional `name`, `page` and `page_size`.

```json
{ "name": "Server" }
```

Layout ids differ per Hudu instance — resolve them by name, never hardcode. This tool has been observed to return 25 layouts for `page_size: 1`, so do not rely on `page_size`; keep paging until a page comes back empty.

## Get Single Asset Layout

`hudu_get_asset_layout` takes `id` (required). Call it before writing any asset's `custom_fields`.

**Abridged returned record:**
```json
{
  "id": 5,
  "name": "Server",
  "icon": "fas fa-server",
  "color": "#2196F3",
  "active": true,
  "fields": [
    { "label": "Hostname", "field_type": "Text", "required": true, "position": 1 },
    { "label": "IP Address", "field_type": "Text", "required": false, "position": 2 },
    { "label": "Operating System", "field_type": "Dropdown", "required": false, "position": 3 },
    { "label": "RAM (GB)", "field_type": "Number", "required": false, "position": 4 },
    { "label": "Warranty Expiry", "field_type": "Date", "required": false, "position": 5 },
    { "label": "Notes", "field_type": "RichText", "required": false, "position": 6 }
  ]
}
```

## Create / Update Asset Layout

`hudu_create_asset_layout` takes `name` (required) and optional `fields` (array), `icon`, `color`, `icon_color`, `active`, `include_comments`, `include_files`, `include_passwords` and `include_photos`. `hudu_update_asset_layout` takes `id` (required) and any of the same fields.

```json
{
  "name": "Network Switch",
  "icon": "fas fa-network-wired",
  "color": "#4CAF50",
  "active": true,
  "fields": [
    { "label": "IP Address", "field_type": "Text", "required": true, "position": 1 },
    { "label": "Model", "field_type": "Text", "required": false, "position": 2 },
    { "label": "Firmware Version", "field_type": "Text", "required": false, "position": 3 },
    { "label": "Port Count", "field_type": "Number", "required": false, "position": 4 },
    { "label": "Location", "field_type": "Text", "required": false, "position": 5 },
    { "label": "Managed", "field_type": "CheckBox", "required": false, "position": 6 }
  ]
}
```

## Relations

`hudu_list_relations` takes only `page` and `page_size` — it has no entity filter. To find an asset's relations, page through all relations and keep those whose `fromable_id` or `toable_id` matches the asset's id (with the matching `fromable_type` / `toable_type`). No tool creates a relation.

## Passwords Linked to an Asset

Asset credentials are reached through the password tools, not through an asset tool. Create one linked to an asset with `hudu_create_asset_password`, passing `company_id`, `name`, `passwordable_type: "Asset"` and `passwordable_id` set to the asset's id; only create sets the link. See the `hudu-passwords` skill.

## Not Served

No tool unarchives an asset, uploads files, or attaches photos to an asset.
