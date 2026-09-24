# Hudu Passwords Error Reference

Errors reach the agent as a tool error string, not an HTTP status.

## Failure modes

| Condition | What the tool returns | Resolution |
|-----------|-----------------------|------------|
| Password access disabled on the server's API key (every other Hudu tool works) | `Authentication failed - invalid API key` | The key is valid; Hudu answers 401 `Bad credentials` for a key lacking password permission and the server maps every 401 to this message. Report that credential access is not enabled for this connection. Do not report the key as expired or ask for it to be rotated |
| The API key itself is invalid (every Hudu tool fails) | `Authentication failed - invalid API key` | Report that the Hudu MCP server's connection is failing; nothing in this plugin can read or change its key |
| Insufficient permission | `Access forbidden - insufficient permissions` | Report the denied operation; an administrator changes the key's permissions in Admin > API Keys |
| Deletion disabled on the API key | A tool error from `hudu_delete_asset_password` | Deletion is a per-key toggle; leave the record in place |
| Unknown or deleted password id | `Resource not found` | Verify the id with `hudu_list_asset_passwords` |
| Missing `name` or `company_id`, or an invalid value | `Validation error` | Provide the required arguments |
| Invalid `password_folder_id` | `Validation error` | Take the folder id from an existing record's `password_folder_id`, or omit it; no tool lists password folders |
| Rate limited | `Rate limit exceeded and max retries reached` | The server already retried; wait before retrying |
| Hudu server failure | `Server error: <status>` | The server retried once already; retry later |

## Secure Error Handling

1. When a password tool fails, report the condition from the table above, never the arguments you sent (they may include a password value).
2. On `Authentication failed - invalid API key` from a password tool, check whether another Hudu tool (for example `hudu_test_connection`) succeeds before deciding which row applies.
3. On `Resource not found`, report that the password was not found and stop; do not retry other ids speculatively.
