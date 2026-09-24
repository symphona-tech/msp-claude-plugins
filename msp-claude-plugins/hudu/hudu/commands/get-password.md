---
description: Retrieve a password from Hudu (with security logging)
argument-hint: "<name> <company> [type] [show]"
arguments: [name, company, type, show]
---

# Get Hudu Password

Retrieve a password from Hudu. Company is required for security.

The Hudu MCP server must be connected, with password access enabled on its API key. Company is required.

## Security Notice

**All password access is logged in Hudu's activity logs.**

When you retrieve a password, the following is recorded:
- API key used to access the password
- Timestamp of access
- Action performed (view, update, etc.)

An operator reviews that log in the Hudu web UI; no tool in this plugin reads it.

**NEVER include actual password values in summaries, reports, or logs.** Both `hudu_list_asset_passwords` and `hudu_get_asset_password` return the plaintext `password` (and `otp_secret`), so every tool result in this command is sensitive, masked output included.

## Steps

1. **Validate parameters**
   - Ensure company is provided
   - Resolve the company name to an id with `hudu_list_companies` (`name`)
   - Validate password search term

2. **Search for password**
   - Call `hudu_list_asset_passwords` with `company_id` and `name`
   - Page through `page` 1, 2, 3… until a page returns fewer items than `page_size` or none; "Found N …" counts one page only
   - If `type` is given, keep only records whose `password_type` matches (no tool argument filters by type)

3. **Display results**
   - Show password details (name, username, URL)
   - Mask password by default, even though the list result already contains the value
   - Reveal password only with --show flag: for a single match, call `hudu_get_asset_password` with its `id` and show the plaintext `password` it returns

## Tools

1. `hudu_list_companies` — resolve the company to `company_id`
2. `hudu_list_asset_passwords` — find matching records (`company_id`, `name`, `page`)
3. `hudu_get_asset_password` — only with `show`, for the single chosen record (`id`)

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| name | string | Yes | - | Password name (partial match) |
| company | string | Yes | - | Company name (required) |
| type | string | No | - | Password type filter |
| show | boolean | No | false | Show actual password value |

## Examples

### Search for Password

```
/get-password "Domain Admin" --company "Acme Corp"
```

### Show Password Value

```
/get-password "Domain Admin" --company "Acme Corp" --show
```

### Filter by Type

```
/get-password "firewall" --company "Acme Corp" --type "Network"
```

### Search by Partial Name

```
/get-password "admin" --company "Acme Corp"
```

## Output

### Password Found (Masked)

```
Found 1 password matching "Domain Admin" in Acme Corporation

Password: Domain Admin - ACME
------------------------------------------------------------
Company:       Acme Corporation
Type:          Administrative
Folder:        Infrastructure > Domain Controllers

Username:      administrator@acme.local
Password:      **************
URL:           https://dc01.acme.local

Description:
Primary domain administrator account. Use for:
- Domain controller management
- Group Policy changes
- AD user management

Last Updated:  2025-11-15
------------------------------------------------------------

To reveal password: /get-password "Domain Admin" --company "Acme Corp" --show

Note: Accessing passwords is logged for security audit.
```

### Password Found (Revealed)

```
Found 1 password matching "Domain Admin" in Acme Corporation

Password: Domain Admin - ACME
------------------------------------------------------------
Company:       Acme Corporation
Type:          Administrative
Folder:        Infrastructure > Domain Controllers

Username:      administrator@acme.local
Password:      SecureP@ssw0rd123!
URL:           https://dc01.acme.local

Description:
Primary domain administrator account. Use for:
- Domain controller management
- Group Policy changes
- AD user management

Last Updated:  2025-11-15
------------------------------------------------------------

WARNING: This access has been logged to Hudu's activity logs.
```

### Multiple Matches

```
Found 3 passwords matching "admin" in Acme Corporation

+----------------------------+------------------------+-----------------+--------------+
| Name                       | Username               | Type            | Last Updated |
+----------------------------+------------------------+-----------------+--------------+
| Domain Admin - ACME        | administrator@acme...  | Administrative  | 2025-11-15   |
| Local Admin - Servers      | .\Administrator        | Administrative  | 2025-12-01   |
| Firewall Admin             | admin                  | Network         | 2025-10-10   |
+----------------------------+------------------------+-----------------+--------------+

Refine search:
  /get-password "Domain Admin" --company "Acme Corp"
  /get-password "admin" --company "Acme Corp" --type "Network"
```

### No Results

```
No passwords found matching "xyz" in Acme Corporation

Suggestions:
  - Check spelling of the password name
  - Try a partial name match
  - Remove type filter to broaden search
  - Check if password exists in Hudu

Example searches:
  /get-password "admin" --company "Acme Corp"
  /get-password "domain" --company "Acme Corp"
```

## Type Reference

### Common Password Types

| Type | Description |
|------|-------------|
| Administrative | Admin/root credentials |
| Application | Software logins |
| Network | Network device access |
| Service Account | Automated accounts |
| User | End-user credentials |
| Vendor | Third-party access |
| Cloud | Cloud service credentials |

Note: Password types are custom per Hudu instance.

## Error Handling

### Company Required

```
Error: Company is required for password lookup

For security, you must specify a company:
  /get-password "Domain Admin" --company "Acme Corp"

This ensures passwords are accessed in proper context.
```

### No Results

```
No passwords found matching "invalid" in Acme Corporation

Suggestions:
  - Verify the password name
  - Check if password exists in Hudu
  - Try a partial match

Example:
  /get-password "admin" --company "Acme Corp"
```

### Invalid Company

```
Company not found: "Acm"

Did you mean?
  - Acme Corporation
  - Acme East Division

Try: /get-password "Domain Admin" --company "Acme Corporation"
```

### Access Denied

When `hudu_list_asset_passwords` or `hudu_get_asset_password` returns `Authentication failed - invalid API key` while other Hudu tools work, the key is valid and password access is disabled on it (Hudu answers 401 for a key lacking password permission). Do not report the key as invalid or expired.

```
Access denied to passwords

Credential access is not enabled for this Hudu connection.
Contact your Hudu administrator to enable password access
for the MCP server's API key in Admin > API Keys.
```

### API Error

When every Hudu tool fails (for example `hudu_test_connection` also returns `Authentication failed - invalid API key`, or tools return `Server error: <status>`), the problem is the MCP server's connection, which nothing in this plugin can read or change.

```
Error reaching Hudu

The Hudu MCP server's connection is failing.
Ask the operator who runs the Hudu MCP server to check it.

Retry later.
```

## Security Best Practices

1. **Always specify company** - Prevents accidental access to wrong company
2. **Use specific names** - Avoid overly broad searches
3. **Review activity logs** - Regularly check password access logs in the Hudu web UI (no tool here reads them)
4. **Don't screenshot** - Avoid capturing revealed passwords
5. **Close after use** - Clear terminal after accessing sensitive data
6. **Verify need** - Only access passwords when necessary
7. **Never share output** - Password values must not appear in summaries or reports

## Related Commands

- `/lookup-asset` - Find related asset
- `/search-articles` - Find related documentation
- `/find-company` - Verify company details
