# Hudu Asset Field Reference

## Core Asset Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | integer | System | Auto-generated unique identifier |
| `company_id` | integer | Yes | Parent company |
| `asset_layout_id` | integer | Yes | The layout (template) this asset uses |
| `name` | string | Yes | Asset display name |
| `primary_serial` | string | No | Primary serial number |
| `primary_model` | string | No | Primary model name |
| `primary_mail` | string | No | Primary email |
| `archived` | boolean | No | Whether the asset is archived |
| `slug` | string | System | URL-friendly identifier |
| `object_type` | string | System | Always "Asset" |

## Custom Fields (Dynamic)

Custom fields are returned on read in a `fields` array, where each entry is a key-value pair defined by the asset layout.

On write, `hudu_create_asset` and `hudu_update_asset` take `custom_fields` as an object with the layout's field labels as keys. Read the layout with `hudu_get_asset_layout` first to learn the labels and which are required; do not invent keys.

## Metadata Fields

| Field | Type | Description |
|-------|------|-------------|
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last update timestamp |
| `url` | string | Full URL to asset in Hudu |
| `asset_layout_name` | string | Name of the asset layout (read-only) |
| `company_name` | string | Name of the parent company (read-only) |

## Asset Layout Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | integer | System | Auto-generated unique identifier |
| `name` | string | Yes | Layout name (e.g., "Server") |
| `icon` | string | No | Font Awesome icon class |
| `color` | string | No | Hex color code |
| `icon_color` | string | No | Icon color |
| `active` | boolean | No | Whether layout is active |
| `sidebar_folder_id` | integer | No | Sidebar folder |
| `fields` | array | Yes | Array of field definitions |
