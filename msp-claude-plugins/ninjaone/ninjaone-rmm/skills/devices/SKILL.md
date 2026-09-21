---
name: "NinjaOne Devices"
description: >
  NinjaOne device management through the plugin's MCP tools: device lookup and
  listing, active alerts, the per-device activity log, Windows service
  inventory, custom fields, and reboot — plus health-check workflows for
  Windows, Mac, and Linux endpoints running the NinjaRMM agent.
when_to_use: >-
  When querying or acting on NinjaRMM-enrolled endpoints — inventory lookups, service
  state, alert review, or reboots. Use when: ninjaone device, ninjarmm
  device, ninja device list, device inventory ninja, ninja services,
  device reboot ninja, or ninja endpoint.
---

# NinjaOne Device Management

Manage NinjaRMM-enrolled endpoints: look devices up, review their alerts and activity, inspect Windows services, and reboot safely.

Every operation below is an MCP tool call. The plugin does not build HTTP requests and holds no NinjaOne credential — the MCP server authenticates and this skill names the tool to call.

## Anti-triggers

- **A device managed by a different RMM** — the endpoint has to be running
  the NinjaRMM agent. Use `atera-devices`, `ncentral-devices`,
  `datto-rmm-devices`, `syncro-assets`, or
  `connectwise-automate-computers` for those fleets. "Device", "endpoint",
  and "agent" mean the same thing in all six; only the enrolment differs.
- **What a device is alerting on** — this skill covers the endpoint's own
  state; the alert queue and its severity model are `ninjaone-alerts`.
- **Every device belonging to a client** — that is an organization-scoped
  query; use `ninjaone-organizations`.
- **The documented record of a machine rather than the live one** — an
  asset record in a documentation platform is not the RMM's view of the
  endpoint; use `hudu-assets` or `itglue-configurations`.

## Core operations

| Tool | What it does |
|---|---|
| `ninjaone_devices_list` | List devices. Filter by `organization_id`, `device_class`, or `online`. Pages by `cursor` |
| `ninjaone_devices_get` | One device by `device_id` |
| `ninjaone_devices_alerts` | Active alerts for a device, optionally filtered by `severity` |
| `ninjaone_devices_activities` | The per-device activity log, optionally filtered by `activity_type` |
| `ninjaone_devices_services` | Windows services on a device, optionally filtered by `state` |
| `ninjaone_devices_get_custom_fields` | Read a device's custom field values |
| `ninjaone_devices_update_custom_fields` | Write a device's custom field values |
| `ninjaone_devices_reboot` | Schedule a reboot. Destructive — see below |

Key fields on a device record: `id`, `systemName`, `offline`, `lastContact`, `os.name`, `nodeRoleId`, `organizationId`, `policyId`.

## Listing is cursor-paged

`ninjaone_devices_list` returns a `cursor` when more devices remain. Page until it stops coming back. A single response is one page, never the whole fleet, and treating it as a complete estate is the most common way a cross-organization sweep reports a wrong total.

## Reboot

`ninjaone_devices_reboot` takes `device_id` and an optional `reason`.

> **Destructive operation.** There is no undo, and the endpoint is unreachable for the minutes that follow. Always confirm the device is online first — see the safe-reboot workflow below.

**The tool exposes no reboot mode.** NinjaOne's API distinguishes a graceful reboot that notifies the logged-in user from a forced one that does not; this tool does not take that argument, so the distinction is not selectable from here. Do not tell a user which mode will be used.

## What this tool surface does not cover

These are NinjaOne platform capabilities with no tool in this plugin. There is no HTTP fallback, so a request for one is answered by saying it is unavailable rather than by constructing a call.

- **Windows service control.** `ninjaone_devices_services` lists services and their state; it cannot start, stop, or restart one.
- **Device record updates.** Renaming a device or changing its role or policy has no tool. Only custom fields are writable.
- **Maintenance windows.** Scheduling or cancelling maintenance has no tool.
- **Hardware and software inventory.** Disks, volumes, processors and installed software have no tools; `ninjaone_devices_get` returns the device record rather than its inventory.
- **Device approval.** Changing a device's approval mode has no tool.
- **Script and command execution.** No tool in this plugin runs anything on an endpoint.

## Workflows

### Safe reboot with validation

```text
1. ninjaone_devices_get      { device_id }   → assert "offline": false
2. ninjaone_devices_alerts   { device_id }   → review severity; abort if critical
3. ninjaone_devices_reboot   { device_id, reason }
4. Re-call ninjaone_devices_get every 30s (up to 10 min) → wait for "offline": false
5. If still offline after 10 min → ninjaone_devices_alerts for new alerts; escalate
```

**Error recovery**: if step 1 shows `"offline": true`, do not reboot. Check `lastContact` and alerts to diagnose. A control call against an offline endpoint fails late rather than early.

### Check server health

```text
1. ninjaone_devices_get        { device_id }              → confirm online, note OS and role
2. ninjaone_devices_alerts     { device_id }              → triage by severity
3. ninjaone_devices_services   { device_id, state }       → verify critical services are RUNNING
4. ninjaone_devices_activities { device_id, limit }       → check for recent relevant events
```

Disk and volume free space is not available through this surface, so a health check built here covers alerts, services and activity rather than capacity.

## Best practices

1. **Check `offline` before rebooting** — control requests to offline devices fail, and retrying queues the same failure.
2. **Give `ninjaone_devices_reboot` a `reason`** — it is the only context the activity log will carry about why the machine went down.
3. **Poll after a reboot** — confirm the device came back at 30s intervals before reporting success.
4. **Page `ninjaone_devices_list` to exhaustion** — one page is not the fleet.

## Reference

See [REFERENCE.md](./REFERENCE.md) for device roles, device classes, and the fields these tools return.

## Related Skills

- [Organizations](../organizations/SKILL.md) — Organization management
- [Alerts](../alerts/SKILL.md) — Alert monitoring
