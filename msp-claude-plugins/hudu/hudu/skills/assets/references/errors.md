# Hudu Assets Error Reference

## Failure modes

Errors reach the agent as tool error strings, not HTTP statuses.

| Condition | What the tool returns | Resolution |
|-----------|-----------------------|------------|
| Missing `name`, `company_id` or `asset_layout_id`, or a layout-required custom field not supplied | `Validation error` | Supply the required arguments; call `hudu_get_asset_layout` and provide every field with `required: true` |
| The asset or layout id does not exist, or was deleted | `Resource not found` | Verify the id with `hudu_list_assets` or `hudu_list_asset_layouts` |
| The server's API key lacks permission for the operation (for example, deletion is disabled for the key) | `Access forbidden - insufficient permissions`, or another tool error | Report that the operation is not enabled for this connection; archive instead only if that is really intended, since archiving is one-way through this plugin |
| Every tool fails with an authentication error | `Authentication failed - invalid API key` | The server's API key itself is the problem; report it to whoever operates the Hudu MCP server. Nothing in this plugin can read or change the key |
| Rate limited | `Rate limit exceeded and max retries reached` | The server already retried; wait before retrying |
| Hudu server error | `Server error: <status>` | The server already retried once; retry later |

## Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Name required | Missing name | Pass `name` |
| Company required | No company_id | Pass `company_id` |
| Layout required | No asset_layout_id | Pass `asset_layout_id` |
| Invalid layout | Bad asset_layout_id | Call `hudu_list_asset_layouts` first |
| Required field missing | Layout requires a field | Check layout fields and provide required ones |

## Error Recovery Pattern

1. If `hudu_create_asset` returns `Validation error`, call `hudu_get_asset_layout` with the `asset_layout_id` you used.
2. List the entries in `fields` with `required: true`, compare their labels with the `custom_fields` keys you sent, and retry with the missing ones supplied — or ask the user for the missing values.
