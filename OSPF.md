# FortiGate 60F — Standalone OSPF Lab with a Cisco Router

| | |
|---|---|
| Devices | FortiGate 60F and Cisco router or Layer-3 switch |
| Firmware | FortiOS 7.0.x |
| Purpose | Form an Area 0 neighbor and exchange LAN routes |
| License | No paid FortiGuard subscription required |
| Est. time | 30–45 minutes |

## Overview

This lab teaches OSPF on a plain routed Ethernet link. It does not use IPsec. After this adjacency works, `Site-to-Site-VPN.md` applies the same neighbor and route concepts to a tunnel interface.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives student PC `10.61.10.10`.

## Topology

```text
FortiGate PC LAN                OSPF transit                 Cisco LAN
10.~~.10.0/24                                                    10.~~.20.0/24
PC 10.~~.10.10
     |                                                               |
[ FortiGate ] internal1 10.~~.255.1/30 ─── 10.~~.255.2/30 [ Cisco router ]
 router-id 10.~~.255.1                              router-id 10.~~.255.2
                         Area 0.0.0.0
```

| Network | Owner | Advertised by |
|---|---|---|
| `10.~~.10.0/24` | FortiGate side | FortiGate |
| `10.~~.255.0/30` | Transit | Both |
| `10.~~.20.0/24` | Cisco side | Cisco |

## Prerequisites

- A dedicated routed FortiGate interface; remove it from a hardware switch if necessary
- Cisco interface configured as a Layer-3 port or SVI
- No duplicate IP addressing
- Firewall policies for the later transit test, with NAT off

## Configuration

### Step 1 — Address and test the transit link

Configure `internal1` as `10.~~.255.1/255.255.255.252` and enable Ping under **Network > Interfaces**. Configure the Cisco side as `10.~~.255.2/30`, then confirm both routers can ping one another before enabling OSPF.

### Step 2 — Configure FortiGate OSPF

If **Network > OSPF** is hidden, enable Advanced Routing under **System > Feature Visibility**. In **Network > OSPF**:

1. Set Router ID to `10.~~.255.1`.
2. Create Area `0.0.0.0`.
3. Add `10.~~.255.0/255.255.255.252` to Area `0.0.0.0`.
4. Add `10.~~.10.0/255.255.255.0` to the same area.
5. Create an OSPF interface bound to `internal1`, network type Broadcast, cost `10`.
6. Make the user-LAN interface passive if it must be advertised but should never form neighbors.

Equivalent FortiGate CLI:

```shell
config router ospf
    set router-id 10.~~.255.1
    config area
        edit 0.0.0.0
        next
    end
    config ospf-interface
        edit "to-cisco"
            set interface "internal1"
            set network-type broadcast
            set cost 10
        next
    end
    config network
        edit 1
            set prefix 10.~~.255.0 255.255.255.252
            set area 0.0.0.0
        next
        edit 2
            set prefix 10.~~.10.0 255.255.255.0
            set area 0.0.0.0
        next
    end
end
```

The network entry activates OSPF on matching interfaces and advertises the prefix. Use passive-interface controls for interfaces where advertisements are needed but Hellos are not.

> **Screenshot:** Network > OSPF showing router ID, Area 0, and the two networks.

### Step 3 — Configure Cisco OSPF

```text
configure terminal
router ospf 1
 router-id 10.~~.255.2
 passive-interface default
 no passive-interface <transit-interface>
 network 10.~~.255.0 0.0.0.3 area 0
 network 10.~~.20.0 0.0.0.255 area 0
end
```

Replace `<transit-interface>` with the real Cisco interface. The Cisco wildcard masks are not FortiGate subnet masks.

### Step 4 — Permit transit traffic

Create policies for the actual interface pair under **Policy & Objects > Firewall Policy**. For a two-way ping test, allow both FortiGate-LAN-to-transit and transit-to-FortiGate-LAN directions. Keep NAT off. OSPF packets to the FortiGate control plane do not require a transit policy, but routed user traffic does.

## Verification

```shell
get router info ospf neighbor
get router info ospf interface
get router info ospf database brief
get router info routing-table ospf
get router info routing-table details 10.~~.20.10
```

Expected result: the Cisco peer appears in `Full` state and `10.~~.20.0/24` appears as an OSPF route through `10.~~.255.2`.

On Cisco:

```text
show ip ospf neighbor
show ip route ospf
show ip ospf interface
```

Finally, test from an endpoint on each LAN. Router-originated pings do not prove the firewall policies or endpoint gateways.

The standard FortiGate-side student PC is `10.~~.10.10/24`; its gateway is normally `10.~~.10.4` when following `VLAN.md`.

## Cost and passive interfaces

OSPF selects the path with the lowest accumulated cost. Changing FortiGate interface cost from `10` to `100` makes this link less attractive only when another OSPF path exists. A passive interface advertises its connected network without attempting neighbor adjacency, reducing noise and avoiding accidental peers on user networks.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No neighbor appears | Transit IP connectivity is broken, OSPF not enabled on the link, or interface is passive | Ping both transit IPs and compare network statements/passive settings |
| Neighbor remains `Init` | One-way Hello communication | Check VLANs, ACLs, multicast handling, and captures in both directions |
| Neighbor cycles through `ExStart`/`Exchange` | MTU or duplicate router-ID problem | Match MTU and assign unique stable router IDs |
| Neighbor stops at `2-Way` on broadcast Ethernet | It is a DROther relationship | This can be normal between DROthers; with only two routers they should normally reach Full |
| Neighbor is Full but remote LAN route is absent | Remote LAN is not advertised or filtered | Check Cisco network statement and OSPF database |
| Route exists but hosts cannot communicate | Missing firewall policy, endpoint gateway, or return path | Test hop by hop and keep NAT off |
| Adjacency never forms after an authentication change | Authentication type/key mismatch | Configure identical OSPF authentication on both ends or remove it for the beginner lab |

Useful capture:

```shell
diagnose sniffer packet internal1 'ip proto 89' 4 20 l
```

## Notes

- OSPF uses IP protocol 89, not TCP or UDP port 89.
- The router ID need not be an interface address, but it must be unique and stable.
- Do not run OSPF on an untrusted WAN for this lab.
