# FortiGate 60F — FortiClient Remote-Access IPsec VPN

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Give remote FortiClient users routed access to the lab LAN |
| License | No paid FortiGuard subscription required |
| Est. time | 25–40 minutes |

## Overview

This lab builds a dial-up, route-based IPsec VPN. The FortiGate does not know the client's public address in advance, so it accepts an authenticated FortiClient on `wan1`, assigns an address from a private pool, and installs split routes for the lab LAN.

This differs from `SSLVPN.md`: IPsec uses IKE/IPsec (normally UDP 500 and 4500), a pre-shared key, and phase 1/phase 2 negotiation. SSL VPN uses TLS on the configured SSL-VPN port and an SSL-VPN portal. Both can use the same local users, but their tunnel objects, client profiles, and firewall policies are separate.

> **Day 1 Cisco prerequisite:** To include the student PC VLAN in remote access, configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) first, then confirm the FortiGate route to `10.~~.10.0/24` through `10.~~.~~.4`.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives student PC `10.61.10.10`, LAN `10.61.61.0/24`, and WAN `200.0.0.61`.

## Prerequisites

- FortiClient VPN installed on a computer outside the FortiGate LAN
- A reachable public or lab-routed address on `wan1`
- No upstream carrier-grade NAT unless UDP 500/4500 can be forwarded
- LAN `10.~~.~~.0/24` on `internal1`
- A configuration backup

## Network overview

| Item | Value |
|---|---|
| FortiGate WAN | `wan1`, `200.0.0.~~` |
| Protected LAN | `internal1`, `10.~~.~~.0/24` |
| VPN name | `RA-IPsec` |
| User group | `IPsec-Users` |
| Client pool | `10.~~.250.10–10.~~.250.50` |
| IKE | IKEv2 where the installed FortiClient supports it |
| Tunnel mode | Split tunnel to `10.~~.~~.0/24` |

The standard student PC `10.~~.10.10` is on `10.~~.10.0/24`, separate from the protected LAN in this base example. To reach that PC through IPsec, also include a `PC-NET-~~` object (`10.~~.10.0/24`) in split-tunnel routing. In the Day 1 Cisco design, the VPN-to-PC policy still uses outgoing interface `internal1`, because `COREbaba` routes VLAN 10 behind that link. Confirm the FortiGate route points through `10.~~.~~.4`.

```text
Remote FortiClient ── Internet/lab WAN ── wan1 [ FortiGate ] internal1 ── LAN
                                           200.0.0.~~          10.~~.~~.0/24
```

## Configuration

### Step 1 — Create the user and group

1. Go to **User & Authentication > User Definition** and create a **Local User**.
2. Use a strong lab password and name the user `ipsec-user`.
3. Go to **User & Authentication > User Groups**.
4. Create a Firewall group named `IPsec-Users` and add `ipsec-user`.

> **Screenshot:** User Groups showing `IPsec-Users` and its local member.

### Step 2 — Create the protected-network address

Go to **Policy & Objects > Addresses** and create:

| Field | Value |
|---|---|
| Name | `LAN-~~` |
| Type | Subnet |
| Subnet | `10.~~.~~.0/255.255.255.0` |
| Interface | `any` |

### Step 3 — Run the IPsec wizard

1. Go to **VPN > IPsec Wizard**.
2. Name the tunnel `RA-IPsec`.
3. Choose **Remote Access** and **FortiClient**.
4. Set Incoming Interface to `wan1`.
5. Select pre-shared-key authentication, enter a long random key, and select `IPsec-Users`.
6. Set Local Interface to `internal1` and Local Address to `LAN-~~`.
7. Set the client range to `10.~~.250.10–10.~~.250.50` with mask `255.255.255.255` if offered by the wizard.
8. Enable IPv4 split tunnel and keep `LAN-~~` as the protected destination.
9. Select IKEv2 if the FortiOS 7.0 wizard and installed FortiClient expose the same option. If not, keep the wizard's compatible default; both ends must use the same IKE version.
10. Review the generated tunnel, address pool, route, and firewall policy before saving.

> **Screenshot:** IPsec Wizard review showing `wan1`, `IPsec-Users`, the client pool, and `LAN-~~`.

> Gotcha: a phase 1 pre-shared key proves that the endpoint knows the shared secret; the username and password identify the individual user. A mismatch in either stage prevents the connection.

### Step 4 — Review phase 1 and phase 2

Go to **VPN > IPsec Tunnels**, edit `RA-IPsec`, and confirm:

- Remote Gateway is Dialup User.
- Interface is `wan1`.
- The IKE version matches FortiClient.
- NAT traversal is enabled when the client may be behind NAT.
- User authentication references `IPsec-Users`.
- Mode configuration assigns the client pool.
- Phase 2 covers the protected LAN and client address space created by the wizard.

Do not casually change proposals after exporting or building the FortiClient profile. Phase 1 and phase 2 proposals must overlap on both ends.

### Step 5 — Review the firewall policy

The wizard should create a policy similar to:

| Field | Value |
|---|---|
| Incoming Interface | `RA-IPsec` |
| Outgoing Interface | `internal1` |
| Source | generated client-pool object and/or `IPsec-Users` |
| Destination | `LAN-~~` |
| Schedule | `always` |
| Service | `ALL` for the first test |
| Action | Accept |
| NAT | Off when LAN devices route replies to the FortiGate |
| Logging | All Sessions |

NAT is normally off because the client pool is routed through the FortiGate. If Cisco `COREbaba` has no route back to `10.~~.250.0/24`, add `ip route 10.~~.250.0 255.255.255.0 10.~~.~~.1`. Using NAT is a workable lab shortcut, but it hides the real VPN client address from LAN devices.

### Step 6 — Configure FortiClient

In FortiClient, open **Remote Access**, add an IPsec VPN, and use:

| Field | Value |
|---|---|
| Remote Gateway | `200.0.0.~~` |
| Authentication Method | Pre-shared Key |
| Pre-shared Key | Same key as phase 1 |
| IKE version/proposals | Match the FortiGate |
| Username | `ipsec-user` |

If the FortiGate tunnel uses a specific peer ID, enter that value as FortiClient's Local ID. Save, disconnect the test computer from the lab LAN, and connect from a different routed network.

## Verification

1. Confirm the client receives an address from `10.~~.250.10–50`.
2. Confirm its route table contains `10.~~.~~.0/24` through the VPN while its default route remains local.
3. Ping a reachable LAN host, not only the FortiGate itself.
4. In **Log & Report > VPN Events**, check the successful user and tunnel name.

```shell
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list
get router info routing-table all
```

The tunnel should have an active IKE gateway, an up selector, an assigned client IP, and increasing packet counters.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No response from gateway | WAN address is unreachable, UDP 500/4500 is blocked, or upstream NAT is missing forwarding | Test from a genuinely remote network and verify the upstream path and NAT-T |
| IKE negotiation fails immediately | Pre-shared key, IKE version, proposal, or peer ID mismatch | Compare phase 1 and FortiClient settings exactly |
| Username is repeatedly rejected | User is not in `IPsec-Users`, password is wrong, or user authentication method differs | Test the local account, group membership, and generated phase 1 settings |
| Tunnel connects but no client address appears | Mode configuration/client pool is missing or overlaps another subnet | Restore the generated pool and use a unique subnet |
| Client reaches the FortiGate but not LAN hosts | Missing tunnel-to-LAN policy, wrong policy destination, host firewall, or no return route | Check policy counters, test a known host, and route the client pool back through the FortiGate |
| Client internet stops after connection | Split tunneling is disabled or a default route was pushed | Limit the protected network to `LAN-~~` and reconnect |
| Tunnel is up but packets do not increase | Client route points elsewhere or phase 2 selectors do not cover the traffic | Inspect the client route table and both phase 2 selectors |

For IKE troubleshooting during one controlled connection attempt:

```shell
diagnose debug reset
diagnose vpn ike log-filter clear
diagnose debug application ike -1
diagnose debug enable
```

Stop immediately after the test:

```shell
diagnose debug disable
diagnose debug reset
```

## Notes

- FortiGate-generated pings are local-out traffic and do not prove that the tunnel-to-LAN firewall policy works.
- A paid FortiGuard subscription is not required for local-user dial-up IPsec. FortiClient VPN-only functionality is sufficient for this lab.
- Use unique credentials and certificates in production; a shared PSK is intentionally simple for a beginner lab.
