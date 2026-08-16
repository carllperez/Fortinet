# FortiGate 60F — LDAP with Windows Active Directory

| | |
|---|---|
| Device | FortiGate 60F and Windows Server Active Directory |
| Firmware | FortiOS 7.0.x |
| Purpose | Authenticate an AD group in a firewall policy |
| License | No paid FortiGuard subscription required |
| Est. time | 35–50 minutes |

## Overview

FortiGate LDAP integration binds to Active Directory, searches for users/groups, and validates passwords. In this lab an AD group is used in a firewall policy. This is prompted firewall authentication; use `FSSO.md` when users should be identified transparently from Windows logons.

All domain values below are examples.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives Windows client `10.61.10.10`.

## Topology and example values

```text
Windows client 10.~~.10.10 ── PC-VLAN10 [ FortiGate ] ── Windows Server / Domain Controller
```

| Item | Example |
|---|---|
| Windows client | `10.~~.10.10/24`, gateway `10.~~.10.4` |
| Domain | `lab.example` |
| NetBIOS name | `LAB` |
| Domain controller | `10.~~.~~.10` |
| Base DN | `DC=lab,DC=example` |
| Common Name Identifier | `sAMAccountName` |
| Bind account | `LAB\svc_fgt_ldap` |
| Allowed AD group | `CN=FortiGate-Internet,OU=Groups,DC=lab,DC=example` |

## Prerequisites

- FortiGate can route to and resolve/reach the domain controller
- AD user and group already exist
- A least-privilege bind account can read the required directory tree
- Time is synchronized
- LDAP TCP 389 is reachable for the lab; use LDAPS/StartTLS with a trusted CA in production

## Configuration

### Step 1 — Create the LDAP server object

1. Go to **User & Authentication > LDAP Servers**.
2. Click **Create New**.
3. Configure:

| Field | Value |
|---|---|
| Name | `LAB-AD` |
| Server IP/Name | `10.~~.~~.10` |
| Server Port | `389` for this isolated lab |
| Common Name Identifier | `sAMAccountName` |
| Distinguished Name | `DC=lab,DC=example` |
| Bind Type | Regular |
| Username | `LAB\svc_fgt_ldap` or its full bind DN |
| Password | Bind-account password |

4. Use **Test Connectivity** and **Test User Credentials** where exposed.
5. Browse the directory tree and confirm the intended group is visible.

> **Screenshot:** LDAP server object with successful connection and the directory tree visible.

> Gotcha: `sAMAccountName` lets users enter short names such as `student1`. Using `cn` often fails when the user's display/common name differs from the logon name.

### Step 2 — Create the remote user group

1. Go to **User & Authentication > User Groups**.
2. Create a Firewall group named `AD-FortiGate-Internet`.
3. Under Remote Groups, select LDAP server `LAB-AD`.
4. Select the exact AD group `FortiGate-Internet` rather than matching every LDAP user.
5. Save.

### Step 3 — Use the group in a policy

Under **Policy & Objects > Firewall Policy**, create a narrow authenticated internet policy:

| Field | Value |
|---|---|
| Incoming Interface | `PC-VLAN10` |
| Outgoing Interface | `wan1` |
| Source Address | `10.~~.10.0/24` |
| Source User/Group | `AD-FortiGate-Internet` |
| Destination | `all` |
| Service | `DNS`, `HTTP`, `HTTPS` |
| NAT | On |
| Logging | All Sessions |

Place it above any broad unauthenticated allow policy. When a new supported connection is attempted, FortiGate should present authentication and validate the user against AD.

## Verification

Test directly from the CLI:

```shell
diagnose test authserver ldap LAB-AD student1 <password>
```

The result should report successful authentication and the expected group membership. Because the command line can expose the entered password in screen history, use a temporary lab credential and clear the terminal where appropriate.

Then authenticate from the Windows client and check **Log & Report > User Events** and **Forward Traffic** for the username and policy.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Connection test times out | Routing, server firewall, port, or source path is wrong | Ping the DC, capture TCP 389, and verify the return route |
| Bind succeeds but user test fails | Base DN or Common Name Identifier is wrong | Use `DC=lab,DC=example` and `sAMAccountName` for AD logon names |
| User authenticates but policy denies | User is not in the selected AD group or policy is shadowed | Check returned groups and policy order |
| Directory tree cannot be browsed | Bind account lacks read access or bind username format is invalid | Test the exact bind DN/domain account and permissions |
| Works by IP but not DC hostname | DNS is wrong on FortiGate | Configure system DNS/domain resolution or keep the lab server IP |
| LDAP works but user is prompted repeatedly | Browser/authentication session behavior or policy mismatch | Check user-event logs and ensure subsequent traffic matches the same authenticated policy |
| LDAPS fails after enabling it | FortiGate does not trust the DC certificate chain or name | Import/trust the issuing CA and use a matching server name |

For one test only:

```shell
diagnose sniffer packet any 'host 10.~~.~~.10 and port 389' 4 20 l
```

## Notes

- Plain LDAP exposes credentials to anyone able to observe the path. Keep it isolated for the lab and prefer LDAPS/StartTLS in production.
- LDAP group authentication is not FSSO; it does not automatically map workstation IPs to logged-on users.
- The bind account should not be a Domain Admin.
