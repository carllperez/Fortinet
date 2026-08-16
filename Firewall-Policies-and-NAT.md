# FortiGate 60F — Firewall Policies and NAT Fundamentals

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Permit, log, translate, and then restrict PC-to-WAN traffic |
| License | No paid FortiGuard subscription required |
| Est. time | 25–35 minutes |

## Overview

A FortiGate policy answers: which traffic may enter on this interface and leave on that interface? Policies are evaluated in displayed order, from top to bottom. The first complete match determines the action. If no policy matches, traffic is denied implicitly.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

```text
PC 10.~~.10.10 ── PC-VLAN10 [ FortiGate ] wan1 ── upstream gateway
```

## Network overview

| Item | Value |
|---|---|
| PC network | `10.~~.10.0/24` |
| Student PC | `10.~~.10.10/24`, gateway `10.~~.10.4` |
| FortiGate PC gateway | `10.~~.10.4` on `PC-VLAN10` |
| WAN | `200.0.0.~~/24` on `wan1` |
| WAN gateway | `200.0.0.1` |

This guide uses the PC VLAN from `VLAN.md`. If you are building the firewall-policy lab first, create `PC-VLAN10` and place the PC-facing switch port in VLAN 10 before testing.

## Policy fields in plain language

| Field | What it matches or does |
|---|---|
| Incoming/Outgoing Interface | The actual traffic direction through the FortiGate |
| Source/Destination | Address objects or groups |
| Schedule | When the rule is active |
| Service | IP protocol and ports, such as DNS or HTTPS |
| Action | Accept or deny |
| NAT | Translates the source for accepted traffic |
| Logging | Records starts/ends or all allowed sessions |

## Configuration

### Step 1 — Create address and service objects

Under **Policy & Objects > Addresses**, create:

- `PC-NET-~~`: `10.~~.10.0/24`
- `Lab-PC`: `10.~~.10.10/32`

Create an address group named `Trusted-Lab` and add `Lab-PC`. Under **Policy & Objects > Services**, inspect the built-in `DNS`, `HTTP`, and `HTTPS` objects. Create a custom service only when the required protocol/port does not already exist.

### Step 2 — Create the initial internet policy

Go to **Policy & Objects > Firewall Policy**, click **Create New**, and set:

| Field | Value |
|---|---|
| Name | `PC-to-WAN` |
| Incoming Interface | `PC-VLAN10` |
| Outgoing Interface | `wan1` |
| Source | `PC-NET-~~` |
| Destination | `all` |
| Schedule | `always` |
| Service | `ALL` for the first connectivity test |
| Action | Accept |
| NAT | Enabled, Use Outgoing Interface Address |
| Log Allowed Traffic | All Sessions |

> **Screenshot:** New Policy showing `PC-VLAN10` to `wan1`, NAT enabled, and logging enabled.

Outgoing-interface NAT changes `10.~~.10.10` to the address on `wan1` and tracks a translated source port. Replies are mapped back to the original PC session.

### Step 3 — Test and inspect logs

Browse from the PC, then open **Log & Report > Forward Traffic**. Confirm the source, destination, policy name/ID, service, action, bytes, and translated source fields.

### Step 4 — Restrict the policy

After basic connectivity works:

1. Change Source from `PC-NET-~~` to `Trusted-Lab` if only that PC should browse.
2. Change Service from `ALL` to `DNS`, `HTTP`, and `HTTPS`.
3. Save and start new test sessions.
4. Confirm web and DNS work, while an unlisted service is denied.

For an explicit logged denial, create a deny policy below the narrow allow rule for `PC-NET-~~` to `wan1`, enable logging, and keep the implicit deny concept in mind. Do not place a broad allow above the narrow restriction.

## Policy order example

| Order | Policy | Result |
|---:|---|---|
| 1 | Trusted PC: DNS/HTTP/HTTPS | Narrow traffic accepted |
| 2 | LAN explicit deny and log | Other LAN traffic denied and visible |
| implicit | No matching policy | Denied |

Moving the broad rule to the top would shadow the narrow rule.

## Verification

```shell
show firewall policy
get router info routing-table details 8.8.8.8
diagnose sniffer packet any 'host 10.~~.10.10' 4 20 l
```

For a targeted flow trace:

```shell
diagnose debug reset
diagnose debug flow filter saddr 10.~~.10.10
diagnose debug flow trace start 20
diagnose debug enable
```

Generate one connection, then stop with:

```shell
diagnose debug disable
diagnose debug reset
```

Look for the selected policy ID, route, and SNAT decision.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Policy shows zero bytes | Incoming/outgoing interface, source, destination, service, or policy order does not match | Use a flow trace and compare every packet field |
| FortiGate can browse/ping but PC cannot | Local-out traffic does not use the transit policy | Fix the `PC-to-WAN` policy, client gateway, and NAT |
| PC reaches IP addresses but not names | DNS service is missing or client DNS is wrong | Permit DNS to the configured resolver and renew client settings |
| Private source appears on WAN | NAT is off or a central-NAT design overrides expectations | Enable policy NAT using outgoing-interface address for this lab |
| Restriction has no effect | Older broad policy is above it or existing sessions remain | Move the narrow policy upward and create new sessions |
| Inter-LAN traffic unexpectedly breaks after enabling NAT | NAT hides the original source and return design was already routed | Disable NAT on private routed policies |

## Notes

- A route decides where traffic could go; a firewall policy decides whether it may go.
- Policy NAT is source NAT. A VIP in `VIP-Port-Forwarding.md` performs destination NAT.
- Security profiles that depend on FortiGuard updates are outside this subscription-free fundamentals lab.
