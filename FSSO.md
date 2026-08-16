# FortiGate 60F — Fortinet Single Sign-On with Active Directory

| | |
|---|---|
| Device | FortiGate 60F and Windows Active Directory |
| Firmware | FortiOS 7.0.x |
| Purpose | Map Windows logons to source IPs for identity-based policy |
| License | No paid FortiGuard subscription required for local polling |
| Est. time | 40–60 minutes |

## Overview

Fortinet Single Sign-On (FSSO) learns that a user logged onto a Windows workstation, maps the username and AD groups to the workstation IP, and lets a firewall policy match that identity without prompting again.

This guide uses FortiGate local AD polling because it requires no Windows Collector Agent software. It also explains Collector Agent and Domain Controller Agent modes so the deployment choices are clear.

> **Day 1 Cisco prerequisite:** If the Windows client uses `10.~~.10.10/24`, configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) first. The FortiGate must have a working route back to VLAN 10.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives Windows client `10.61.10.10`.

## Components and software requirements

| Mode | External software | Behavior |
|---|---|---|
| FortiGate local polling | None | FortiGate polls each DC's Windows security events and queries LDAP groups |
| Collector Agent polling | FSSO Collector Agent on Windows | Collector polls DCs and sends logon records to FortiGate |
| DC Agent mode | Collector Agent plus DC Agent on each domain controller | DC Agent feeds logons to Collector; DC reboot/install change is required |

Collector/DC Agent installers must be obtained from an authorized Fortinet download source and matched to the supported environment. Download entitlement may depend on the Fortinet account; local polling avoids that dependency for this lab.

## Prerequisites

- Working `LDAP-Active-Directory.md` configuration
- Domain-joined Windows client `10.~~.10.10/24`, gateway `10.~~.10.4`
- Domain controller `10.~~.~~.10`
- Cisco `COREbaba` routes VLAN 10 to FortiGate `internal1`, and the FortiGate has a route back through `10.~~.~~.4`
- A test AD group such as `FortiGate-Internet`
- FortiGate polling credentials permitted to read required Windows security log/RPC information and LDAP group membership
- Windows audit/logon events enabled and time synchronized

## Configuration: local FortiGate polling

### Step 1 — Confirm LDAP

Under **User & Authentication > LDAP Servers**, confirm `LAB-AD` connects and can browse the intended group. FSSO logon discovery without group lookup is not enough for a group-based policy.

### Step 2 — Create the AD polling connector

1. Go to **Security Fabric > External Connectors**.
2. Click **Create New**.
3. Under Endpoint/Identity, choose **Poll Active Directory Server**.
4. Name it `LAB-AD-Polling` and set server `10.~~.~~.10`.
5. Enter the polling account and password.
6. Select LDAP server `LAB-AD`.
7. Under Users/Groups, add only the required `FortiGate-Internet` group.
8. Save and wait for a green/connected status.

> **Screenshot:** External Connectors showing the local FSSO agent and connected AD polling connector.

For more than one DC, create a connector for each DC that can authenticate users.

### Step 3 — Create the FSSO user group

1. Go to **User & Authentication > User Groups**.
2. Create `FSSO-Internet`.
3. Set Type to **Fortinet Single Sign-On (FSSO)**.
4. Add the AD `FortiGate-Internet` group exposed by the connector.

### Step 4 — Create the identity policy

Under **Policy & Objects > Firewall Policy**, create:

| Field | Value |
|---|---|
| Incoming Interface | `internal1` |
| Outgoing Interface | `wan1` |
| Source Address | `10.~~.10.0/24` |
| Source User/Group | `FSSO-Internet` |
| Destination | `all` |
| Service | `DNS`, `HTTP`, `HTTPS` |
| NAT | On |
| Logging | All Sessions |

Place this above a broad unauthenticated allow policy. Log off and back onto the domain from the test workstation so a fresh event is generated.

The client is behind the Cisco VLAN 10 SVI, but its traffic reaches the FortiGate on `internal1`; a policy using `PC-VLAN10` will not match the Day 1 routed-core topology.

## Collector Agent mode (alternative)

For larger or busier AD environments, install the Windows FSSO Collector Agent and choose either polling or DC Agent mode during its configuration. Configure group filters in the Collector Agent, set a communication password, then on FortiGate create an **FSSO Agent on Windows AD** connector pointing to the Collector Agent (default communication is commonly TCP 8000). Select the previously configured LDAP server for advanced group lookup.

DC Agent mode installs a component on each domain controller and normally requires a DC reboot. It can capture logons more directly but adds software to critical servers. It is not necessary for this small lab.

## Verification

```shell
diagnose debug authd fsso list
diagnose debug authd fsso server-status
diagnose debug fsso-polling detail 1
```

Expected result:

- Connector/server status is connected.
- The logged-on workstation IP appears with the AD username.
- `MemberOf` includes the selected AD group.
- Forward traffic logs include that username and the FSSO policy.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Connector is down | Credentials, RPC/security-log access, routing, or Windows firewall is wrong | Validate the polling account and DC connectivity |
| Connector is up but user list is empty | No fresh domain logon event, cached credentials, or unsupported/missed event | Log off/on while connected to the DC and inspect polling detail |
| User appears but has no group | LDAP lookup failed or group filter excludes it | Test `LAB-AD` and add the exact group to the connector filter |
| User/group is correct but policy does not match | Workstation IP changed, policy order is wrong, or source interface differs | Renew identity, check the IP mapping, and inspect policy counters |
| Shared PC shows the previous user | Stale FSSO session/logoff detection | Log off cleanly, shorten timeout carefully, or clear only the stale lab record |
| Many simultaneous logons are missed | Local polling cannot keep up with events | Consider Collector Agent/DC Agent mode |
| Laptop uses cached domain credentials off-network | No new DC event exists | Reconnect to the domain and perform a real domain logon |

## Notes

- FSSO identifies a user/IP relationship; it does not authenticate every packet.
- NAT devices between clients and FortiGate can collapse many users behind one IP and break identity accuracy.
- Grant the polling/bind accounts only the permissions required for event and group lookup.
