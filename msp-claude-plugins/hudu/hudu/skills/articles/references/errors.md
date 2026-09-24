# Hudu Articles Error Reference

## Failure modes

Errors reach the agent as a tool error string, not an HTTP status.

| Condition | What the tool returns | Resolution |
|-----------|-----------------------|------------|
| `name` missing on create, or a value is invalid | `Validation error` | Provide article name and check argument values |
| The server's Hudu API key is rejected (every tool fails) | `Authentication failed - invalid API key` | Report that the Hudu MCP server's key is not accepted; the plugin holds no key to check |
| The key lacks a permission, such as article deletion | `Access forbidden - insufficient permissions` (or another tool error) | Report that the permission is not enabled for this connection |
| The article id does not exist or was deleted | `Resource not found` | Verify article ID with `hudu_list_articles` |
| Too many requests | `Rate limit exceeded and max retries reached` | The server already retried; wait before retrying |
| Hudu server failure | `Server error: <status>` | The server already retried once; retry later |

## Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Name required | Missing name | Add `name` to the tool arguments |
| Invalid folder | Bad folder_id | Call `hudu_list_folders` first |
| Invalid company | Bad company_id | Call `hudu_list_companies` first |

## Error Recovery Pattern

1. Call `hudu_create_article`.
2. If it returns `Validation error` and a `folder_id` was passed, the folder may belong to another company or not exist: call `hudu_create_article` again without `folder_id` to create the article at root level, and tell the user it was not filed.
3. Any other error is reported as returned; do not retry blindly.
