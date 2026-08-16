# FortiGate 60F — Dual-WAN SD-WAN Load Balancing and Failover

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.15 |
| Purpose | Load balancing and automatic failover between PLDT and DITO |
| License | Not required |
| Est. time | 30–45 minutes |

## Overview

This guide configures a FortiGate 60F to use two DHCP internet connections in an SD-WAN zone. New sessions are distributed across PLDT and DITO while both links are healthy. If either ISP fails its health check, the FortiGate removes that path and sends new traffic through the surviving ISP.

The downstream wireless router could not be modified, so it remains in router mode. Its WAN interface is statically configured as `10.28.0.13` with gateway `10.28.0.1`, while its client network remains `192.168.0.0/24`. This produces double NAT but supports outbound internet, load balancing, and failover.

> This guide records the actual working homelab values. Change the addresses and interface names when reusing it in another environment.

> **Topology note:** This SD-WAN lab is a separate working homelab using the FortiGate `internal` hardware switch and downstream router. It does not require `COREtaas-~~` or `COREbaba-~~` unless you intentionally adapt its LAN side to the Day 1 Cisco topology.

## Prerequisites

- FortiGate 60F running FortiOS 7.0.15
- Administrator access to the FortiGate GUI and CLI
- PLDT connected to FortiGate `wan1`
- DITO connected to FortiGate `wan2`
- Both ISP interfaces configured to obtain their addresses through DHCP
- Downstream router connected from its WAN/Internet port to FortiGate physical LAN2 (`internal2`)
- A configuration backup before changing interfaces, routes, or firewall policies

## Network overview

| Item | Value |
|---|---|
| PLDT interface | wan1 (DHCP) |
| PLDT gateway observed in lab | 192.168.1.1 |
| DITO interface | wan2 (DHCP) |
| DITO gateway observed in lab | 201.0.0.1 |
| SD-WAN zone | virtual-wan-link |
| FortiGate LAN interface | internal hardware switch |
| FortiGate LAN IP | 10.28.0.1/24 |
| FortiGate DHCP pool | 10.28.0.100–10.28.0.200 |
| Downstream router WAN | 10.28.0.13 (static) |
| Downstream router gateway | 10.28.0.1 |
| Downstream router LAN | 192.168.0.0/24 |
| Performance SLA | Internet_Health |
| Health-check targets | 1.1.1.1 and 8.8.8.8 |

## Topology

```text
PLDT modem LAN ──> FortiGate wan1 (DHCP) ┐
                                           ├──> virtual-wan-link (SD-WAN)
DITO modem LAN ──> FortiGate wan2 (DHCP) ┘
                                                     |
FortiGate LAN2/internal2 <── internal 10.28.0.1/24 ──┘
           |
           └──> Router WAN 10.28.0.13 (static)
                    |
                    └──> Router LAN/Wi-Fi 192.168.0.0/24
```

## Configuration

### Step 1 — Configure the WAN interfaces

1. Go to Network > Interfaces.
2. Edit `wan1`.
3. Set Alias to `PLDT`.
4. Set Addressing Mode to DHCP.
5. Set Role to WAN.
6. Set Distance to `10`.
7. Allow Ping for troubleshooting; do not expose HTTPS or SSH unless explicitly required.
8. Click OK.
9. Repeat the process for `wan2`, using Alias `DITO`, DHCP addressing, and Distance `10`.
10. Confirm both interfaces are up and have received an address and gateway.


<!-- Paste screenshot here: Step 1a — wan1/PLDT configured for DHCP with distance 10 -->


<img width="1436" height="781" alt="Screenshot 2026-07-22 at 2 58 36 PM" src="https://github.com/user-attachments/assets/14cf3ede-4f52-4f89-aaaf-a3d88500da66" />


<!-- Paste screenshot here: Step 1b — wan2/DITO configured for DHCP with distance 10 -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 2 57 25 PM" src="https://github.com/user-attachments/assets/56163d38-45ae-4f88-b338-5663bdca7862" />


<!-- Paste screenshot here: Step 1c — Interface list showing wan1 and wan2 up -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 2 59 05 PM" src="https://github.com/user-attachments/assets/c0683255-00cb-45b8-9785-14caf95d8853" />

> DHCP interfaces normally install routes with a distance of 5. Using distance 10 on both WAN interfaces keeps the two links consistent with the SD-WAN default route.

### Step 2 — Configure the internal transit network

1. Go to Network > Interfaces.
2. Edit the `internal` hardware-switch interface.
3. Set Addressing Mode to Manual.
4. Set IP/Netmask to `10.28.0.1/255.255.255.0`.
5. Enable HTTPS, SSH, and Ping for internal management as required.
6. Enable the DHCP server.
7. Set the address range to `10.28.0.100` through `10.28.0.200`.
8. Set Netmask to `255.255.255.0`.
9. Set Default Gateway to Same as Interface IP.
10. Set DNS Server to Same as System DNS.
11. Click OK.

<!-- Paste screenshot here: Step 2a — internal interface set to 10.28.0.1/24 -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 00 09 PM" src="https://github.com/user-attachments/assets/f5452f3d-7a09-4197-99c7-f42a2c6d7bb0" />

<!-- Paste screenshot here: Step 2b — internal DHCP pool set to 10.28.0.100–10.28.0.200 -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 00 25 PM" src="https://github.com/user-attachments/assets/9181e86d-5c6d-45e2-b312-b0d52698dda0" />


> The physical LAN ports are members of the `internal` hardware switch and share one broadcast domain. The router is physically connected to LAN2/internal2, but policies use the logical `internal` interface.
>
> Changing the internal address disconnects the current GUI session. Renew the management computer's DHCP lease, or temporarily use `10.28.0.10/24` with gateway `10.28.0.1`, then reconnect at `https://10.28.0.1`.

### Step 3 — Add the WAN interfaces to SD-WAN

1. Go to Network > SD-WAN > SD-WAN Zones.
2. Click Create New > SD-WAN Member.
3. Configure the first member:

| Field | Value |
|---|---|
| Interface | wan1 (PLDT) |
| SD-WAN Zone | virtual-wan-link |
| Gateway | 0.0.0.0 / automatic because the interface uses DHCP |
| Cost | 0 |
| Priority | 1 |
| Weight | 1 |
| Status | Enabled |

4. Save the member.
5. Repeat for `wan2` (DITO) using the same cost, priority, and weight.
6. Confirm both members appear under `virtual-wan-link`.

<!-- Paste screenshot here: Step 3a — Create SD-WAN member for wan1/PLDT -->

<!-- Paste screenshot here: Step 3b — Create SD-WAN member for wan2/DITO -->

<!-- Paste screenshot here: Step 3c — virtual-wan-link containing both SD-WAN members -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 02 02 PM" src="https://github.com/user-attachments/assets/65e308df-03db-4482-a473-b93eae110e71" />


> An interface must be removed from other configuration references before FortiGate will offer it as an SD-WAN member. In this lab, `wan1` was referenced by the existing outbound firewall policy. `wan2` was added first, the policy's outgoing interface was changed from `wan1` to `virtual-wan-link`, and then `wan1` became available to add.

### Step 4 — Create the Performance SLA

1. Go to Network > SD-WAN > Performance SLAs.
2. Click Create New.
3. Configure:

| Field | Value |
|---|---|
| Name | Internet_Health |
| Protocol | Ping |
| Servers | 1.1.1.1 and 8.8.8.8 |
| Participants | wan1 and wan2 |
| Check interval | 1000 ms |
| Failures before inactive | 5 |
| Restore link after | 5 |
| Update static route | Enabled |

4. Enable SLA Target and configure:

| Metric | Threshold |
|---|---|
| Latency | 250 ms |
| Jitter | 50 ms |
| Packet loss | 10% |

5. Click OK.
6. Wait for both participants to show alive and begin reporting latency, jitter, and packet loss.

<!-- Paste screenshot here: Step 4a — Internet_Health servers, participants, and link-status settings -->

<!-- Paste screenshot here: Step 4b — SLA target thresholds -->

<!-- Paste screenshot here: Step 4c — Both Performance SLA participants alive -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 03 17 PM" src="https://github.com/user-attachments/assets/415863a9-2541-493c-98f9-639de5712c5e" />

> `Internet_Health` is the Performance SLA name and is also selected as the Required SLA Target in the SD-WAN rule. There is no separate built-in object named `Internet_SLA`.

### Step 5 — Create the load-balancing rule

1. Go to Network > SD-WAN > SD-WAN Rules.
2. Click Create New.
3. Configure:

| Field | Value |
|---|---|
| Name | Internet_Load_Balance |
| Source | all |
| Destination | all |
| Strategy | Maximize Bandwidth (SLA) |
| Interface preference | wan1 and wan2 |
| Required SLA target | Internet_Health |
| Status | Enabled |

4. Save the rule.
5. Keep it above the implicit/default SD-WAN rule.

<!-- Paste screenshot here: Step 5a — Internet_Load_Balance rule settings -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 03 50 PM" src="https://github.com/user-attachments/assets/0aaa7a8a-fa2e-40e2-a191-a62b8f6ce26c" />

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 04 02 PM" src="https://github.com/user-attachments/assets/4e5d8201-2bf0-4b94-b262-c7f5f96df131" />

<!-- Paste screenshot here: Step 5b — Rule list showing Internet_Load_Balance above the implicit rule -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 04 23 PM" src="https://github.com/user-attachments/assets/7810ae63-3872-4291-b4a1-ca81b1fc192d" />


> FortiOS 7.0.15 uses round-robin for Maximize Bandwidth rules created in the GUI. This distributes new sessions across both healthy links. Avoid source-IP-only balancing in this topology because every downstream client is NATed behind the router's single WAN address, `10.28.0.13`.

### Step 6 — Create the SD-WAN default route

1. Go to Network > Static Routes.
2. Click Create New.
3. Set Destination to Subnet.
4. Set the destination to `0.0.0.0/0.0.0.0`.
5. Set Interface to `virtual-wan-link`.
6. Leave Gateway blank or unspecified.
7. Set Distance to `10`.
8. Enable the route and click OK.

<!-- Paste screenshot here: Step 6 — Default route through virtual-wan-link -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 04 43 PM" src="https://github.com/user-attachments/assets/3b153023-9f5f-4c43-bbce-ffcb9c7027cd" />

> Do not add separate manually configured default routes for `wan1` and `wan2`. SD-WAN selects the healthy member for the default route.

### Step 7 — Configure the outbound firewall policy

1. Go to Policy & Objects > Firewall Policy.
2. Create a new policy or edit the original internal-to-WAN policy.
3. Configure:

| Field | Value |
|---|---|
| Name | Internal_to_SD-WAN |
| Incoming Interface | internal |
| Outgoing Interface | virtual-wan-link |
| Source | all |
| Destination | all |
| Schedule | always |
| Service | ALL |
| Action | Accept |
| NAT | Enabled |
| IP Pool Configuration | Use Outgoing Interface Address |
| Log Allowed Traffic | All Sessions |

4. Enable the policy and click OK.
5. Avoid creating a duplicate policy if the existing outbound policy already has these values.

<!-- Paste screenshot here: Step 7a — Internal_to_SD-WAN firewall policy settings -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 08 36 PM" src="https://github.com/user-attachments/assets/8e7db0e9-2c3f-4a3b-9017-0bfee1534f4e" />

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 08 52 PM" src="https://github.com/user-attachments/assets/42319c19-fd88-4ce4-968c-6c89cd7acd02" />


<!-- Paste screenshot here: Step 7b — Firewall policy list showing the enabled SD-WAN policy -->

<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 09 19 PM" src="https://github.com/user-attachments/assets/64157594-2477-4c73-b7d9-57b445d335e8" />


### Step 8 — Connect the downstream router

Use this physical connection:

```text
FortiGate LAN2/internal2 ──> downstream router WAN/Internet port
```

The downstream router retains its existing configuration:

| Setting | Value |
|---|---|
| WAN IP | 10.28.0.13 |
| WAN gateway | 10.28.0.1 |
| LAN subnet | 192.168.0.0/24 |
| Operating mode | Router mode |

No router configuration change is required. Client devices continue receiving `192.168.0.x` addresses from the downstream router.

<!-- Paste screenshot here: Step 8 — Downstream client with a 192.168.0.x address and working internet -->

## Verification

### Verify the router and routes

Run:

```shell
get system arp
```

The output should include the downstream router:

```text
10.28.0.13    <router-mac>    internal
```

Check SD-WAN health and selection:

```shell
diagnose sys sdwan health-check Internet_Health
diagnose sys sdwan service
get router info routing-table all
```

Confirm:

1. Both WAN members show `alive`.
2. Both members pass the SLA and are selected by `Internet_Load_Balance`.
3. A default route is present through SD-WAN.
4. A phone connected to the downstream router retains a `192.168.0.x` address and can browse the internet.

<!-- Paste screenshot here: Verification — SD-WAN monitor showing both members healthy and carrying sessions -->

<!-- Paste screenshot here: Verification — Routing table or CLI health-check output -->
<img width="1436" height="781" alt="Screenshot 2026-07-22 at 3 10 11 PM" src="https://github.com/user-attachments/assets/6e7d2711-937c-4638-ad4b-b315d5a85c4e" />

### Test failover

1. Confirm both SD-WAN members are healthy.
2. Disconnect PLDT from FortiGate `wan1`.
3. Wait approximately 5–10 seconds.
4. Open a new website from a downstream client; it should use DITO.
5. Reconnect PLDT and wait for it to become healthy.
6. Disconnect DITO from FortiGate `wan2`.
7. Open a new website; it should use PLDT.
8. Reconnect DITO and confirm both members recover.

> Existing downloads, calls, VPNs, and other sessions can reset during failover because the public source IP changes. New sessions should recover automatically.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `wan1` or `wan2` is missing from the SD-WAN member list | The interface is referenced by a policy, route, VPN, VIP, or another object | Click the interface Ref. count or run `diagnose sys cmdb refcnt show system.interface.name wan1`; migrate the reference to `virtual-wan-link`, then add the member |
| FortiGate internet works, but the downstream router has no internet | Router WAN addressing does not match the FortiGate transit subnet | Check ARP and DHCP traffic; match the FortiGate internal gateway to the router's authorized static WAN configuration |
| Router does not appear in the DHCP widget | The router WAN uses a static address and does not request DHCP | Use ARP capture to discover the router's existing address and expected gateway; do not assume it is a DHCP client |
| Management GUI disconnects after changing `internal` | The management computer is still using the old subnet | Renew DHCP or temporarily configure `10.28.0.10/24`, gateway `10.28.0.1`, and reconnect to `https://10.28.0.1` |
| One or both SLA members show dead | Health-check targets are unreachable through that ISP | Test each WAN path, verify its DHCP address/gateway, and confirm that ping to the health-check targets is allowed |
| Internet works directly from FortiGate LAN but not behind the router | Cable is in the wrong downstream-router port, router WAN is inactive, or router uses unmatched static settings | Connect FortiGate LAN2 to the router WAN/Internet port, verify link status, then capture ARP/DHCP traffic |
| Only one WAN carries sessions while both are healthy | WAN distances, SD-WAN member status, SLA selection, or rule order is incorrect | Set both DHCP WAN distances to 10, verify both members are enabled and pass `Internet_Health`, and place `Internet_Load_Balance` above the implicit rule |

### Discover a static downstream-router WAN address

Check the physical link:

```shell
diagnose hardware deviceinfo nic internal2
```

In the working lab, `link_status` was `Up` at 1000 Mbps/full duplex.

Capture ARP and DHCP traffic while a downstream client tries to browse:

```shell
diagnose sniffer packet internal 'arp or (udp port 67 or 68)' 4 50 l
```

The router repeatedly sent:

```text
arp who-has 10.28.0.1 tell 10.28.0.13
```

<!-- Paste screenshot here: Troubleshooting — Packet capture revealing router WAN 10.28.0.13 and gateway 10.28.0.1 -->

This proved that the router WAN was statically configured as `10.28.0.13` and expected gateway `10.28.0.1`. Changing the FortiGate `internal` address to `10.28.0.1/24` restored connectivity without modifying the router.

## Notes

The downstream router remains in router mode because it is outside the permitted configuration scope. Traffic is NATed once by the downstream router and again by the FortiGate. Outbound browsing, load balancing, and failover work normally, but inbound port forwarding and externally initiated VPNs require configuration on both devices and may also be limited by ISP carrier-grade NAT.

All physical LAN ports that belong to the FortiGate `internal` hardware switch share the `10.28.0.0/24` network. LAN2/internal2 is the physical connection to the router, while firewall policies reference the logical `internal` interface.

Back up the FortiGate configuration after successful testing. FortiOS 7.0.15 should also be reviewed against Fortinet's supported upgrade path and current security guidance before exposing administrative or VPN services to the internet.
