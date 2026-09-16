---
description: View agreement status and entitlements for a company in ConnectWise PSA
argument-hint: "[company_id] [agreement_id] [include_additions] [active_only]"
arguments: [company_id, agreement_id, include_additions, active_only]
---

# Check ConnectWise PSA Agreement

View agreement status, covered products, and remaining hours/incidents for a company.

## Prerequisites

- The `connectwise-manage-mcp` server is configured and reachable
- The server's API member must hold agreement read permissions

## Tools used

| Step | Tool |
|------|------|
| Resolve company | `cw_search_companies` |
| Find company agreements | `cw_search_agreements` |
| Read one agreement | `cw_get_agreement` |
| Read its line items | `cw_get_agreement_additions` |

## Steps

1. **Resolve the company (if named rather than given by ID)**
   - `cw_search_companies` with `conditions=name contains "<company>"`

2. **Retrieve agreements**
   - By ID: `cw_get_agreement` with `id=<agreement_id>`
   - By company: `cw_search_agreements` with
     `conditions=company/id=<company_id> and cancelledFlag=false and (endDate >= [<today>] or endDate = null)`
   - **`cancelledFlag=false` alone is not "active".** A non-cancelled agreement
     whose `endDate` has passed is expired, and reporting it as active
     overstates coverage. An open-ended agreement has a null `endDate`, so both
     clauses are needed.
   - When the caller asked to include expired, drop only the date clauses and
     keep `cancelledFlag=false`; label each result with its `endDate`

3. **Retrieve agreement additions (if include_additions=true)**
   - `cw_get_agreement_additions` with `agreementId=<agreement_id>`
   - These are the line items: what the agreement actually covers and at what
     quantity

4. **Format and return agreement details**
   - Report covered hours, remaining hours and expiry
   - **Name any requested section that is unavailable** rather than returning a
     coverage summary that looks complete

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| company_id | integer | No* | - | Company ID to check |
| agreement_id | integer | No* | - | Specific agreement ID |
| include_additions | boolean | No | true | Include additions |
| active_only | boolean | No | true | Only show active |

*Either company_id or agreement_id is required

## Examples

### Check Company Agreements

```
/check-agreement --company_id 12345
```

### Check Specific Agreement

```
/check-agreement --agreement_id 9876
```

### Include All Agreements (Including Expired)

```
/check-agreement --company_id 12345 --active_only false
```

### Minimal View (No Additions)

```
/check-agreement --agreement_id 9876 --include_additions false
```

## Output

### Company Agreements Summary

```
================================================================================
Agreements for Acme Corporation
================================================================================

Found 2 active agreements

--------------------------------------------------------------------------------
1. Managed Services Agreement (ID: 9876)
--------------------------------------------------------------------------------
Type:           Managed Services
Status:         Active
Start Date:     2025-01-01
End Date:       2026-12-31

Billing:
  Cycle:        Monthly
  Amount:       $2,500.00/month
  Next Invoice: 2026-03-01

Prepaid Hours:
  Purchased:    40.0 hours
  Used:         24.5 hours
  Remaining:    15.5 hours

Coverage:

Additions (3):
  - Azure Management (+$500/month)
  - 24/7 Monitoring (+$300/month)
  - Additional Block Hours - 10 hrs (+$1,250, one-time)

--------------------------------------------------------------------------------
2. Break/Fix Agreement (ID: 9877)
--------------------------------------------------------------------------------
Type:           Time & Materials
Status:         Active
Start Date:     2024-01-01
End Date:       None (Perpetual)

Billing:
  Rate:         $175.00/hour
  Minimum:      0.25 hours

Coverage:

Note: Used for non-covered work.

================================================================================
```

### Detailed Single Agreement

```
================================================================================
Agreement: Managed Services Agreement
================================================================================

Basic Information:
  ID:             9876
  Name:           Managed Services Agreement
  Type:           Managed Services
  Parent:         None

Company:
  Name:           Acme Corporation
  ID:             12345

Status:
  Active:         Yes
  Start Date:     2025-01-01
  End Date:       2026-12-31
  Auto Renew:     Yes
  Days Until Exp: 300 days

================================================================================
Financial Summary
================================================================================

Billing Information:
  Cycle:          Monthly
  Amount:         $2,500.00
  Terms:          Net 30
  Next Invoice:   2026-03-01

Annual Value:     $30,000.00

================================================================================
Prepaid Hours/Block Time
================================================================================

Block Hours Summary:
  Total Purchased:   50.0 hours
  Total Used:        34.5 hours
  Remaining:         15.5 hours

Block Hours Breakdown:
  +40.0 hrs - Initial Agreement Block (Jan 2025)
  +10.0 hrs - Additional Block Purchase (Jan 2026)
  -34.5 hrs - Used to date

Utilization:        69% used

Warning: 15.5 hours remaining. Consider upselling additional block.

================================================================================
Agreement Additions
================================================================================

1. Azure Management
   Type:       Recurring
   Amount:     $500.00/month
   Start:      2025-01-01
   Status:     Active

2. 24/7 Monitoring
   Type:       Recurring
   Amount:     $300.00/month
   Start:      2025-01-01
   Status:     Active

3. Additional Block Hours
   Type:       One-Time
   Amount:     $1,250.00
   Quantity:   10 hours
   Added:      2026-01-15
   Status:     Active

Total Additions: $800.00/month recurring + $1,250.00 one-time

================================================================================
```

## Error Handling

### Missing Required Parameter

```
Error: Either company_id or agreement_id is required

Examples:
  /check-agreement --company_id 12345
  /check-agreement --agreement_id 9876
```

### Company Not Found

```
Error: Company ID 99999 not found

The company may have been deleted or you may not have permission to access it.
Use /search-company to find the correct company ID.
```

### Agreement Not Found

```
Error: Agreement ID 99999 not found

The agreement may have been deleted or you may not have permission to access it.
```

### No Agreements Found

```
No active agreements found for Acme Corporation (ID: 12345)

Options:
- Include expired agreements: /check-agreement --company_id 12345 --active_only false
- Create new agreement in ConnectWise PSA
- Check if company ID is correct
```

### Permission Denied

```
Error: Permission denied

You do not have permission to view agreements.
Contact your ConnectWise administrator.
```

### Expired Agreement Warning

```
Warning: Agreement is expired

Agreement: Managed Services Agreement (ID: 9876)
End Date:  2026-01-31 (4 days ago)

This agreement is no longer active. Time logged may not be covered.

View anyway? [Y/n]
```

### Low Hours Warning

```
Alert: Low prepaid hours remaining

Agreement: Managed Services Agreement
Remaining: 3.5 hours (of 40.0 total)
Utilization: 91%

Recommend contacting customer about purchasing additional hours.
```

### Expiring Soon Warning

```
Alert: Agreement expiring soon

Agreement: Managed Services Agreement
End Date:  2026-02-28 (24 days remaining)
Auto Renew: No

Recommend initiating renewal discussion with customer.
```

## Not available through the tool surface

| Requested | Would need |
|-----------|------------|
| Covered work types | An agreement work type tool |
| Covered work roles | An agreement work role tool |
| Recent usage against the agreement | A way to scope time entries to an agreement. `cw_search_time_entries` charges to a ticket, and this command does not resolve the agreement's tickets |

The original command reported which work types and roles an agreement covers.
**Neither is retrievable**, so a coverage answer built from this command is
about hours and additions only.

This is a real narrowing rather than a cosmetic one: *"is this work covered"*
frequently turns on the work role, and an agreement summary that silently omits
role coverage invites the reader to conclude that covered hours are the whole
answer. Say the role coverage is unavailable whenever billing coverage is the
question being asked.

**Recent usage is available but is not free.** The usage table in the sample
output does not come from any agreement tool — it requires a separate
`cw_search_time_entries` call filtered to the company's tickets. Either make
that call explicitly or omit the section; do not present an agreement response
as though it carried usage.

## Output Includes

- Agreement name, type, and status
- Prepaid hours remaining (for block agreements)
- Incident packs remaining
- Covered configuration types
- Expiration date and billing information
- Agreement additions
- Recent usage summary

## Related Commands

- `/log-time` - Log time against ticket (uses agreement)
- `/close-ticket` - Close ticket with time entry
- `/lookup-config` - View configurations covered by agreement
- `/create-ticket` - Create ticket (checks agreement coverage)
