---
name: "ConnectWise Manage Contacts"
description: >
  ConnectWise PSA contact management: contact records, contact types, communication
  items (email, phone), portal access, and relationships to companies. Essential
  for MSP customer relationship management in ConnectWise PSA.
when_to_use: >-
  When creating, updating, searching, or managing contact records. Use when: connectwise contact,
  contact management, create contact connectwise, contact email, contact phone, customer portal,
  portal access, communication items, contact type, or primary contact.
---

# ConnectWise PSA Contact Management

## Overview

Contacts in ConnectWise PSA represent individuals at your client companies. Contacts can be ticket requestors, agreement signers, project stakeholders, and portal users. This skill covers contact CRUD operations, communication items, contact types, and portal access management.

## Anti-triggers

- **The organisation itself** — company type, status, sites and billing
  defaults hang off the company record, not its contacts; use
  `connectwise-psa-companies`.
- **Your own technicians** — internal staff are `/system/members`, a
  separate entity that contact searches never return; query it through
  `connectwise-psa-api-patterns`.

## Tool surface

Contacts are reached through `connectwise-manage-mcp`, never by direct REST.
The endpoint below is named for orientation only.

```
Base: /company/contacts
```

| Tool | Purpose |
|------|---------|
| `cw_search_contacts` | Search with CW conditions syntax |
| `cw_get_contact` | Fetch one contact by ID |
| `cw_create_contact` | Create a contact |

**There is no contact update tool and no contact delete tool.** This surface is
read-and-create only for contacts — see
[Not available through the tool surface](#not-available-through-the-tool-surface).

## Contact Types

Standard contact types in ConnectWise PSA:

| Type ID | Name | Description |
|---------|------|-------------|
| 1 | Admin | Administrative contact |
| 2 | Primary | Main point of contact |
| 3 | Billing | Billing/accounts payable |
| 4 | Technical | Technical contact |
| 5 | Sales | Sales contact |

**Note:** Contact types are configurable. Query `/company/contacts/types` for your instance's types.

## Complete Contact Field Reference

### Core Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | int | System | Auto-generated unique identifier |
| `firstName` | string(30) | Yes | Contact first name |
| `lastName` | string(30) | No | Contact last name |
| `company` | object | Yes | `{id: companyId}` - Parent company |
| `site` | object | No | `{id: siteId}` - Company site |
| `type` | object | No | `{id: typeId}` - Contact type |

### Contact Information

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string(50) | No | Job title |
| `department` | object | No | `{id: departmentId}` |
| `relationship` | object | No | `{id: relationshipId}` |
| `nickName` | string(30) | No | Nickname/alias |
| `school` | string(50) | No | School/university |
| `marriedFlag` | boolean | No | Marital status |
| `childrenFlag` | boolean | No | Has children |
| `significantOther` | string(30) | No | Spouse/partner name |
| `anniversary` | date | No | Anniversary date |
| `birthDay` | date | No | Birth date |

### Address Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `addressLine1` | string(50) | No | Street address |
| `addressLine2` | string(50) | No | Suite/unit |
| `city` | string(50) | No | City |
| `state` | string(50) | No | State/province |
| `zip` | string(12) | No | Postal code |
| `country` | object | No | `{id: countryId}` |

### Portal Access Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `portalSecurityLevel` | int | No | Portal access level (1-6) |
| `disablePortalLoginFlag` | boolean | No | Disable portal access |
| `unsubscribeFlag` | boolean | No | Opt out of emails |

### Tracking Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inactiveFlag` | boolean | No | Contact is inactive |
| `defaultMergeContactId` | int | No | ID for merge operations |
| `managerContactId` | int | No | Manager contact ID |
| `assistantContactId` | int | No | Assistant contact ID |
| `_info` | object | System | Metadata |

## Communication Items

Communication items store contact methods (email, phone, fax, etc.) for a contact.

### Communication Item Endpoint

```
/company/contacts/{contactId}/communications
```

### Communication Item Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | int | System | Communication ID |
| `type` | object | Yes | `{id: typeId}` - Email, Phone, Fax, etc. |
| `value` | string(250) | Yes | The email/phone/etc. value |
| `extension` | string(15) | No | Phone extension |
| `defaultFlag` | boolean | No | Is primary for this type |
| `communicationType` | string | No | Direct, Fax, Cell, Pager, etc. |

### Communication Types

| Type ID | Name | Description |
|---------|------|-------------|
| 1 | Email | Email address |
| 2 | Phone | Phone number |
| 3 | Fax | Fax number |
| 4 | Cell | Mobile phone |
| 5 | Pager | Pager (legacy) |
| 6 | Direct | Direct line |

### Communication item structure

**No tool adds or edits communication items.** The shape below documents how
ConnectWise models one, for reading a contact record that carries them:

```json
{
  "type": {"id": 1, "name": "Email"},
  "value": "john.smith@acme.com",
  "defaultFlag": true,
  "communicationType": "Email"
}
```

### Phone number structure

A phone number is a communication item whose type is a phone type; the same
read-only constraint applies.

## Portal Access

### Portal Security Levels

| Level | Name | Access |
|-------|------|--------|
| 1 | Admin | Full access to all company tickets/data |
| 2 | Manager | Access to department tickets |
| 3 | User | Access to own tickets only |
| 4 | Limited | View only |
| 5 | Read Only | Read only, no create |
| 6 | Restricted | Minimal access |

### Portal access fields

Portal access is represented on the contact record by `portalSecurityLevel` and
the portal flags documented above. **No tool sets them** — the fields are
readable through `cw_get_contact` and changeable only in the PSA.

### Portal Password Reset

Portal passwords are managed through the ConnectWise portal. The API does not expose password fields.

### Portal Invitation

To invite a contact to the portal:
1. Ensure contact has valid email
2. Set `portalSecurityLevel` > 0
3. Set `disablePortalLoginFlag` = false
4. Portal sends automatic invitation email

## Tool Operations

### Search Contacts

```
cw_search_contacts
  conditions: "company/id=12345 and inactiveFlag=false"
  orderBy:    "lastName asc"
```

### Get a Contact

```
cw_get_contact  id: 67890
```

### Create a Contact

```
cw_create_contact
  firstName: "John"
  lastName:  "Smith"
  companyId: 12345
```

Check the tool's input schema for the fields it accepts. The field reference
above documents the full ConnectWise model, which is **wider than the tool's
arguments**.

### Updating a contact

**Not possible through this surface.** There is no `cw_update_contact`. A
contact's details, portal access and communication items can be read but not
changed from here; corrections are made in the PSA directly.

## Common Query Patterns

**All contacts for a company:**
```
conditions=company/id=12345
```

**Active contacts only:**
```
conditions=inactiveFlag=false
```

**Contacts by type:**
```
conditions=type/id=2
```

**Contacts with portal access:**
```
conditions=portalSecurityLevel>0 and disablePortalLoginFlag=false
```

**Search by name:**
```
conditions=firstName contains "John" or lastName contains "Smith"
```

**Contacts by email:**
```
conditions=communicationItems/value="john@acme.com"
```

**Primary contacts only:**
```
conditions=type/id=2 and inactiveFlag=false
```

## Contact Relationships

### Related Entities

| Entity | Relationship |
|--------|-------------|
| Company | `/company/companies/{companyId}` |
| Tickets | `/service/tickets?conditions=contact/id={id}` |
| Notes | `/company/contacts/{id}/notes` |
| Communications | `/company/contacts/{id}/communications` |
| Groups | `/company/contacts/{id}/groups` |

### Contact Notes

Contacts carry their own note collection in ConnectWise. **No tool reads or
writes it** — `cw_get_ticket_notes` is for service tickets, and there is no
contact equivalent.

## Best Practices

1. **Always include company** - Contacts must belong to a company
2. **Add communication items** - Email/phone essential for notifications
3. **Set contact type** - Helps identify primary contacts
4. **Use portal access wisely** - Grant minimum necessary access
5. **Link to site** - Important for multi-site companies
6. **Avoid duplicates** - Search before creating new contacts

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Company required | Missing company reference | Include `company: {id: x}` |
| firstName required | Missing first name | Provide firstName field |
| Invalid company | Company doesn't exist | Verify company ID |
| Cannot delete | Has related records | Mark as inactive instead |
| Email exists | Duplicate email | Use existing contact |

## Not available through the tool surface

| Operation | Would need |
|-----------|------------|
| Updating a contact | A contact update tool. **None exists** — unlike companies, which have `cw_update_company` |
| Deleting a contact | A delete tool. The surface exposes none at all |
| Adding a communication item (phone, email) | A communications tool |
| Enabling portal access, resetting a portal password, sending an invitation | A portal tool |
| Contact notes | A contact-notes tool |

**Contacts are the most narrowed entity in this plugin.** Companies can be
created and updated; contacts can be created and read only. A workflow that
creates a contact and then corrects it cannot complete here — get the details
right at creation, or make the correction in the PSA.

The communication-item, portal-access and notes sections above are retained as
**domain knowledge** for reading a contact record. They describe how
ConnectWise models this data; they are **not workflows this plugin can
execute.**

## Related Skills