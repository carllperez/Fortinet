# FortiGate 60F — RADIUS Authentication

| | |
|---|---|
| Device | FortiGate 60F and RADIUS server |
| Firmware | FortiOS 7.0.x |
| Purpose | Authenticate a remote user group with Windows NPS or FreeRADIUS |
| License | No paid FortiGuard subscription required |
| Est. time | 30–45 minutes |

## Overview

RADIUS centralizes authentication. FortiGate sends an Access-Request to the server, which returns Access-Accept or Access-Reject and may return group attributes. This lab creates a RADIUS object and user group, tests credentials, and uses the group in an authenticated firewall policy.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives RADIUS server `10.61.61.20`.

## Network overview

| Item | Example |
|---|---|
| RADIUS server | `10.~~.~~.20` |
| Authentication port | UDP `1812` |
| FortiGate RADIUS object | `LAB-RADIUS` |
| FortiGate group | `RADIUS-Users` |
| Shared secret | Unique secret configured identically on both sides |

## Prerequisites

- Windows NPS, FreeRADIUS, or another RADIUS server is installed
- FortiGate's source/NAS IP is registered as a RADIUS client
- Matching shared secret and authentication method are configured
- UDP 1812 is reachable both ways
- A test user is allowed by a RADIUS network policy

## Configuration

### Step 1 — Prepare the RADIUS server

On the RADIUS server:

1. Add the FortiGate interface IP as a RADIUS client/NAS.
2. Enter the same strong shared secret planned for FortiGate.
3. Create a network policy that permits the test user/group.
4. Enable the authentication method required by the FortiGate service. PAP is simple for an isolated test; MS-CHAPv2 is common for services that support it.

RADIUS protects the password using the shared-secret exchange but is not a general encrypted transport. Keep it on a trusted management path or use a secure RADIUS design in production.

### Step 2 — Create the FortiGate RADIUS object

1. Go to **User & Authentication > RADIUS Servers**.
2. Click **Create New**.
3. Set Name `LAB-RADIUS`, server `10.~~.~~.20`, and enter the shared secret.
4. Choose the authentication method expected by the server, or use the service-compatible default.
5. Set NAS IP/source IP when the RADIUS server expects requests from one specific FortiGate address.
6. Use the GUI credential test if available.

> **Screenshot:** RADIUS server object showing successful connectivity and credential test.

### Step 3 — Create the remote group

Go to **User & Authentication > User Groups** and create Firewall group `RADIUS-Users`. Add `LAB-RADIUS` as a remote member.

By default, any valid user accepted by that server can match. To restrict by server-side group, configure a Fortinet group VSA/filter on the RADIUS server and select that exact group in the FortiGate remote-group match.

### Step 4 — Use the group

Use `RADIUS-Users` in a firewall authentication policy as in `LDAP-Active-Directory.md`, or use it for a remote-access VPN that supports RADIUS user authentication. For administrator login, create a remote administrator mapping and least-privilege admin profile separately; do not turn the first RADIUS test account into an unrestricted admin.

## Verification

```shell
diagnose test authserver radius LAB-RADIUS pap student1 <password>
```

Replace `pap` with `chap`, `mschap`, or `mschap2` only when that method is configured end to end. Successful output should say the user authenticated and may list returned group memberships.

Check the RADIUS server's authentication log and FortiGate **Log & Report > User Events**. A server-side log distinguishes Access-Reject from no request received.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Test reports no response | Wrong server IP, routing, UDP 1812, or server firewall | Capture traffic and verify the RADIUS service is listening |
| Server says unknown RADIUS client | FortiGate used a different source IP than the registered NAS | Set/verify source or NAS IP and update the server client entry |
| Server reports bad authenticator/shared secret | Secrets differ | Re-enter the same secret on both sides |
| Access-Reject with valid password | NPS/network policy, group condition, or auth method does not match | Read the server reason code and align the permitted method/group |
| Direct credential test succeeds but FortiGate group fails | Remote group match expects a missing/different VSA | Return the correct Fortinet group attribute or match any user for the first lab |
| VPN/admin service fails but PAP test succeeds | That service negotiates another method or uses a different group | Test the actual method and inspect service-specific logs |

Packet capture:

```shell
diagnose sniffer packet any 'host 10.~~.~~.20 and udp port 1812' 4 20 l
```

For a short authentication debug:

```shell
diagnose debug reset
diagnose debug application fnbamd -1
diagnose debug enable
```

Stop with `diagnose debug disable` and `diagnose debug reset`.

## Notes

- The shared secret is between FortiGate and the RADIUS server; it is not the user's password.
- Authentication (UDP 1812) and accounting (normally UDP 1813) are separate functions.
- RADIUS itself does not require FortiGuard licensing.
