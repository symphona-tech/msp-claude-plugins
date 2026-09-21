# NinjaOne Devices — Reference

Lookup tables for the values the device tools return and accept. Every operation runs through an MCP tool; this file documents data, not requests.

## Device Roles

`nodeRoleId` on a device record.

| Role ID | Name | Description |
|---------|------|-------------|
| 1 | Windows Workstation | Standard Windows endpoint |
| 2 | Windows Server | Windows Server OS |
| 3 | Mac | macOS device |
| 4 | Linux Workstation | Linux desktop |
| 5 | Linux Server | Linux server |

## Device Classes

`device_class` filters `ninjaone_devices_list` and `ninjaone_organizations_devices`. It selects the same distinction the role IDs above describe — workstation versus server, and the operating system family.

## Windows Service States

`state` filters `ninjaone_devices_services`. Services report `RUNNING`, `STOPPED`, `PAUSED`, or a pending transition between them. Match a service by `serviceName`, which is case-sensitive; the display name is not a stable key.

## Alert Severities

`severity` filters `ninjaone_devices_alerts` and `ninjaone_alerts_list`.

| Severity | Meaning |
|---|---|
| `CRITICAL` | Requires immediate attention |
| `MAJOR` | Significant, not immediately service-affecting |
| `MINOR` | Informational or low impact |
| `NONE` | No severity assigned |

## Failure modes

These reach the caller as tool errors rather than as HTTP status codes, but the underlying conditions are the ones to reason about.

| Condition | What it means | Resolution |
|---|---|---|
| Device not found | The `device_id` does not exist, or belongs to an organization this credential cannot reach | Confirm the id with `ninjaone_devices_list` |
| Access denied | The API client's scope does not cover this operation or organization | The credential is scoped at NinjaOne; a read-only client cannot perform a write |
| Conflict | The device is offline or mid-operation | Check `offline` with `ninjaone_devices_get` first; retrying without checking queues the same failure |
| Rate limited | A large sweep exceeded the API's limit partway through | The result is partial. Treat a truncated sweep as incomplete, never as "no further devices" |

## Not available through this plugin

Documented so a reader stops looking rather than assuming the tool exists under another name.

- **Hardware inventory** — disks, volumes, processors.
- **Installed software.**
- **Device approval** — approving or rejecting devices pending enrolment.
- **Device record updates** — renaming, or changing role or policy. Custom fields are the only writable device data.
- **Maintenance windows.**
- **Windows service control** — listing is available; starting, stopping and restarting are not.
