# FortiGate 60F — VLANs with a Cisco Managed Switch

| | |
|---|---|
| Device | FortiGate 60F and managed Cisco IOS switch |
| Firmware | FortiOS 7.0.x |
| Purpose | Build tagged VLAN gateways, DHCP scopes, and controlled inter-VLAN access |
| License | No paid FortiGuard subscription required |
| Est. time | 35–50 minutes |

## Overview

A VLAN separates one physical Ethernet link into multiple Layer-2 broadcast domains. The Cisco uplink carries tagged frames; each FortiGate VLAN interface removes the matching 802.1Q tag, acts as that VLAN's gateway, and applies firewall policy between networks.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives PC `10.61.10.10`, PC VLAN `10.61.10.0/24`, and guest VLAN `10.61.20.0/24`.

## Network overview

| VLAN | Name | FortiGate gateway | DHCP pool | Example switch port |
|---:|---|---|---|---|
| 1 | Transit/native | `10.~~.~~.1/24` on `internal1` | Existing lab scope | Uplink native VLAN |
| 10 | PCs | `10.~~.10.4/24` | `10.~~.10.100–199` | `Gi0/2` access VLAN 10 |
| 20 | Guest | `10.~~.20.1/24` | `10.~~.20.100–199` | `Gi0/3` access VLAN 20 |

The standard student PC address is `10.~~.10.10/24` with gateway `10.~~.10.4`. Configure it statically or create a DHCP reservation for the PC's MAC address.

```text
wan1 [ FortiGate 60F ] internal1 ===== 802.1Q trunk ===== Gi0/1 [ Cisco switch ]
                         VLAN 10 gateway                         Gi0/2 PC
                         VLAN 20 gateway                         Gi0/3 Guest
```

> The existing repository uses `.4` for the PC-network gateway. In this lab the FortiGate owns `10.~~.10.4`, so do not configure a switch SVI with the same IP.

## Prerequisites

- `internal1` is available as the parent interface and physically connected to the switch
- Cisco switch supports 802.1Q trunks
- Management access will not be lost when the trunk changes
- Existing subnets do not overlap VLAN 10 or 20

## Configuration

### Step 1 — Create the VLAN interfaces

1. Go to **Network > Interfaces**.
2. Click **Create New > Interface**.
3. Create `PC-VLAN10` with Type **VLAN**, Interface `internal1`, VLAN ID `10`, and address `10.~~.10.4/255.255.255.0`.
4. Enable Ping for testing. Do not enable WAN-side administrative services.
5. Create `GUEST-VLAN20` on the same parent with VLAN ID `20` and address `10.~~.20.1/255.255.255.0`.

> **Screenshot:** Network > Interfaces showing both VLAN interfaces nested under `internal1`.

> Gotcha: a VLAN ID is a tag, not an IP subnet. VLAN 10 works only when both ends use tag 10; its IP range can be any non-overlapping subnet.

### Step 2 — Enable DHCP per VLAN

Edit each VLAN interface, enable DHCP Server, and set:

| Interface | Range | Gateway | DNS |
|---|---|---|---|
| `PC-VLAN10` | `10.~~.10.100–10.~~.10.199` | Same as interface IP | Same as system DNS |
| `GUEST-VLAN20` | `10.~~.20.100–10.~~.20.199` | Same as interface IP | Same as system DNS |

Use mask `255.255.255.0`. Exclude gateways, switches, servers, and reservations from dynamic ranges.

### Step 3 — Configure the Cisco trunk and access ports

On a typical Cisco IOS switch:

```text
configure terminal
vlan 10
 name PCS
vlan 20
 name GUEST
interface GigabitEthernet0/1
 description TRUNK-TO-FORTIGATE-internal1
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 1,10,20
 no shutdown
interface GigabitEthernet0/2
 description PC-VLAN10
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
interface GigabitEthernet0/3
 description GUEST-VLAN20
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
end
```

Command availability varies by Cisco platform. Verify with:

```text
show interfaces trunk
show vlan brief
```

The native VLAN must agree on both ends. FortiGate traffic on the untagged parent `internal1` corresponds to the Cisco native VLAN; FortiGate VLAN subinterfaces expect tagged frames.

### Step 4 — Create address objects and policies

Create subnet address objects under **Policy & Objects > Addresses** for both VLANs. Then create policies under **Policy & Objects > Firewall Policy**:

| Name | Incoming | Outgoing | Source | Destination | Service | NAT |
|---|---|---|---|---|---|---|
| `PC-to-WAN` | `PC-VLAN10` | `wan1` | `PC-NET` | `all` | `ALL` initially | On |
| `Guest-to-WAN` | `GUEST-VLAN20` | `wan1` | `GUEST-NET` | `all` | `DNS`, `HTTP`, `HTTPS` | On |
| `PC-to-Guest-test` | `PC-VLAN10` | `GUEST-VLAN20` | `PC-NET` | test-host object | `PING` | Off |

Do not create a Guest-to-PC policy. FortiGate is stateful, so replies to sessions initiated by PCs are allowed, but new guest-initiated sessions have no matching policy.

> **Screenshot:** Firewall Policy list showing internet policies and the one narrow inter-VLAN test policy.

## Verification

1. Connect the student PC to `Gi0/2`; it should use `10.~~.10.10/24` with gateway `10.~~.10.4` through a static setting or DHCP reservation.
2. Connect another client to `Gi0/3`; it should receive `10.~~.20.x` with gateway `10.~~.20.1`.
3. Confirm both can reach permitted internet services.
4. Confirm the VLAN 10 PC reaches only the guest host/service explicitly allowed.
5. Confirm a guest client cannot initiate a connection to the PC VLAN.

```shell
show system interface PC-VLAN10
show system interface GUEST-VLAN20
execute ping 10.~~.10.10
diagnose sniffer packet internal1 'vlan 10 or vlan 20' 4 20 l
```

The capture should show tagged traffic on the parent link. Policy counters should increase for the expected interface pair.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Client receives `169.254.x.x` | Access VLAN missing, trunk does not allow the VLAN, or DHCP is disabled | Check Cisco access VLAN, allowed VLAN list, FortiGate VLAN ID, and DHCP scope |
| Untagged management works but VLAN clients do not | Cisco link is access mode or VLAN tags are pruned | Make `Gi0/1` a trunk and allow VLANs 10 and 20 |
| Only one VLAN fails | Tag mismatch between FortiGate and Cisco | Compare FortiGate VLAN ID with `show interfaces trunk` |
| DHCP works but internet does not | Missing VLAN-to-WAN policy, route, or NAT | Check policy incoming interface, default route, and NAT enabled |
| Inter-VLAN ping fails although policy matches | Target host firewall blocks other subnets | Test the gateway first, then allow ICMP on the target host |
| Duplicate-IP warnings or unstable ARP | FortiGate and switch SVI use the same gateway address | Keep the gateway on only one routing device |
| Native VLAN behaves unpredictably | Native VLAN mismatch | Use the same native VLAN on both ends or avoid carrying untagged user traffic |

## Notes

- Routing between directly connected VLAN interfaces still requires a firewall policy.
- NAT should be on for private-to-internet policies and off between normally routed VLANs.
- Complete this guide before `DHCP.md` if you want to build multiple scopes.
