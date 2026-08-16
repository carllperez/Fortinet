# FortiGate 60F — OSPF with Cisco `COREbaba`

| | |
|---|---|
| Devices | FortiGate 60F and Cisco `COREbaba` Layer-3 switch |
| Firmware | FortiOS 7.0.x and Cisco IOS |
| Purpose | Form the Day 1 Area 0 adjacency and advertise the Cisco VLANs to the FortiGate |
| License | No paid FortiGuard subscription required |
| Est. time | 25–40 minutes |

## Overview

This lab follows the OSPF portion of `DAY1-May5-SirRob.txt`. `COREbaba` routes the local VLANs and advertises them to the FortiGate across the routed `10.~~.~~.0/24` link. The FortiGate then knows how to reach the student PC and can carry those learned routes into the Site-to-Site VPN lab.

> **Day 1 Cisco prerequisite:** Configure and verify both `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) first. The VLANs, LACP trunk, SVIs, and routed `COREbaba`-to-FortiGate link must be operational before OSPF is enabled.

> Replace `~~` with the student's monitor number. For monitor 61, the PC is `10.61.10.10/24`, gateway `10.61.10.4`.

## Topology

```text
student PC                    Area 0 adjacency                  FortiGate
10.~~.10.10/24                                                  WAN/VPN
gateway 10.~~.10.4                                                  |
       |                                                            |
[ COREbaba ] Gi0/1 10.~~.~~.4/24 ---- 10.~~.~~.1/24 internal1 [ FortiGate ]
 router-id 10.~~.~~.4                         router-id ~~.0.0.1
```

`COREbaba` advertises these connected Day 1 networks when present: VLAN 1 (`10.~~.1.0/24`), VLAN 10 (`10.~~.10.0/24`), VLAN 50 (`10.~~.50.0/24`), VLAN 100 (`10.~~.100.0/24`), and the routed link.

## Prerequisites

- Complete the routed-core portion of `VLAN.md`.
- FortiGate `internal1` is `10.~~.~~.1/24`.
- `COREbaba` `Gi0/1` is a routed port at `10.~~.~~.4/24`, with `ip routing` enabled.
- Each device can ping the other routed-link address before OSPF is enabled.
- Router IDs are unique.

## Configuration

### Step 1 — Configure OSPF on `COREbaba`

```text
configure terminal
ip routing
router ospf 1
 router-id 10.~~.~~.4
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.~~.0.0 0.0.255.255 area 0
interface GigabitEthernet0/1
 ip ospf network point-to-point
end
```

The broad Cisco wildcard includes the site's `10.~~.x.0/24` connected networks. `passive-interface default` advertises the SVI networks without sending OSPF Hellos to user devices; only `Gi0/1` forms a neighbor.

### Step 2 — Configure OSPF on the FortiGate

If **Network > OSPF** is hidden, enable Advanced Routing under **System > Feature Visibility**. Equivalent CLI:

```shell
config router ospf
    set router-id ~~.0.0.1
    config area
        edit 0.0.0.0
        next
    end
    config ospf-interface
        edit "to-COREbaba"
            set interface "internal1"
            set network-type point-to-point
            set cost 10
        next
    end
    config network
        edit 1
            set prefix 10.~~.~~.0 255.255.255.0
            set area 0.0.0.0
        next
    end
end
```

Both ends use point-to-point network type, matching the supplied Day 1 design. Do not add the Cisco VLANs as FortiGate OSPF networks: they are not FortiGate-connected interfaces; the FortiGate learns them from `COREbaba`.

### Step 3 — Decide how `COREbaba` reaches external networks

For the basic lab, retain the Cisco static default route:

```text
ip route 0.0.0.0 0.0.0.0 10.~~.~~.1
```

This sends internet and unknown destinations to the FortiGate. Advertising a default route through OSPF is an optional advanced design and is not required here.

### Step 4 — Create FortiGate transit policies

OSPF control packets terminate on the FortiGate and do not require a normal forward policy. User traffic does. For PC internet access, create an `internal1` to `wan1` policy with the Cisco VLAN source networks and NAT enabled. For private routed or VPN traffic, use the actual outgoing interface and keep NAT off unless that guide explicitly says otherwise.

## Verification

On the FortiGate:

```shell
get router info ospf neighbor
get router info ospf interface
get router info ospf database brief
get router info routing-table ospf
get router info routing-table details 10.~~.10.10
```

Expected: neighbor `10.~~.~~.4` is Full on `internal1`, and `10.~~.10.0/24` is learned through `10.~~.~~.4`.

On `COREbaba`:

```text
show ip ospf neighbor
show ip ospf interface GigabitEthernet0/1
show ip route ospf
show ip route 0.0.0.0
```

Finally, test from the PC. Router-originated pings alone do not verify the PC gateway, FortiGate policy, or return path.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No neighbor appears | Link IPs, network statement, interface state, or passive setting is wrong | Ping both link addresses and compare OSPF settings |
| Neighbor remains `Init` | One-way Hello communication | Check cabling, ACLs, and packet captures in both directions |
| Neighbor cycles through `ExStart`/`Exchange` | MTU or duplicate router ID | Match MTU and use unique router IDs |
| Neighbor is Full but VLAN routes are absent | SVI is down or not matched by the Cisco network statement | Check `show ip interface brief`, connected routes, and OSPF database |
| FortiGate route points somewhere other than `.4` | A static or competing dynamic route wins | Inspect administrative distance and remove the unintended route |
| Route exists but PC traffic fails | Missing FortiGate policy, wrong PC gateway, or endpoint firewall | Verify gateway `10.~~.10.4`, policy counters, and host firewall |
| Site-to-Site uses WAN instead of tunnel | Competing OSPF path has a lower cost | Follow the tunnel/WAN cost section in `Site-to-Site-VPN.md` |

Useful FortiGate capture:

```shell
diagnose sniffer packet internal1 'ip proto 89' 4 20 l
```

## Notes

- OSPF uses IP protocol 89, not TCP or UDP port 89.
- The router ID need not be an interface address, but it must be unique and stable.
- Keep user-facing SVIs passive; form the adjacency only on the routed FortiGate link.
