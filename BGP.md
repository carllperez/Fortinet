# FortiGate 60F — Basic eBGP Lab

| | |
|---|---|
| Devices | FortiGate 60F and Cisco router or Layer-3 switch |
| Firmware | FortiOS 7.0.x |
| Purpose | Form an eBGP session and exchange two private lab networks |
| License | No paid FortiGuard subscription required |
| Est. time | 30–45 minutes |

## Overview

BGP exchanges IP prefixes between autonomous systems. This lab uses private ASNs: the FortiGate is AS `65001`, the Cisco router is AS `65002`, and they peer across one directly connected `/30` link.

> **Scope:** The supplied Day 1 switching notes use OSPF between FortiGate and Cisco `COREbaba`; they do not define BGP. This is a separate routing exercise with a dedicated Cisco BGP peer. Do not paste this BGP configuration onto `COREbaba` unless the instructor explicitly assigns that role.

> `COREtaas-~~` and `COREbaba-~~` are required only when this BGP lab advertises or tests the Day 1 student-PC VLAN. The standalone `/30` BGP peer itself does not require the collapsed-backbone switches.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives student PC `10.61.10.10`.

## Topology

```text
PC 10.~~.10.10 -- VLAN 10 [ COREbaba ] -- [ FortiGate AS 65001 ]
                                               |
                                      dedicated BGP /30
                                               |
                               [ Cisco BGP peer AS 65002 ] -- 10.~~.20.0/24
```

| Setting | FortiGate | Cisco peer |
|---|---|---|
| Local AS | `65001` | `65002` |
| Router ID | `10.~~.255.1` | `10.~~.255.2` |
| Neighbor | `10.~~.255.2` | `10.~~.255.1` |
| Advertised prefix | `10.~~.10.0/24` | `10.~~.20.0/24` |

The standard FortiGate-side student PC is `10.~~.10.10/24`, using Cisco `COREbaba` gateway `10.~~.10.4` when the design from [VLAN.md](VLAN.md) is used. Before BGP can advertise that exact prefix, the FortiGate must already have `10.~~.10.0/24` in its routing table through OSPF or a static route to `COREbaba`.

## Prerequisites

- The transit interfaces can ping one another
- Each advertised prefix exists in the local routing table
- The dedicated BGP peer link is separate from the FortiGate-to-`COREbaba` Day 1 link
- Transit firewall policies are ready for endpoint testing, with NAT off
- No other router uses AS `65001` or `65002` in the same lab design

## Configuration

### Step 1 — Configure FortiGate BGP

Address a dedicated FortiGate routed interface as `10.~~.255.1/30` and the Cisco peer interface as `10.~~.255.2/30`. Replace `<bgp-interface>` below with that FortiGate interface name; do not reuse `internal1`, which belongs to the `COREbaba` adjacency in the Day 1 topology.

If **Network > BGP** is hidden, enable Advanced Routing in **System > Feature Visibility**. Under **Network > BGP**:

1. Set Local AS to `65001`.
2. Set Router ID to `10.~~.255.1`.
3. Add neighbor `10.~~.255.2` with Remote AS `65002`.
4. Add network `10.~~.10.0/255.255.255.0`.

Equivalent CLI:

```shell
config router bgp
    set as 65001
    set router-id 10.~~.255.1
    config neighbor
        edit "10.~~.255.2"
            set remote-as 65002
        next
    end
    config network
        edit 1
            set prefix 10.~~.10.0 255.255.255.0
        next
    end
end
```

> **Screenshot:** Network > BGP showing AS 65001, the neighbor, and advertised network.

> Gotcha: a BGP `network` statement does not create a route. The exact prefix must already exist in the FortiGate routing table before BGP advertises it.

### Step 2 — Configure the Cisco peer

```text
configure terminal
router bgp 65002
 bgp router-id 10.~~.255.2
 neighbor 10.~~.255.1 remote-as 65001
 network 10.~~.20.0 mask 255.255.255.0
end
```

The Cisco router must also have `10.~~.20.0/24` in its routing table. Exact syntax can differ by Cisco operating system.

### Step 3 — Permit routed test traffic

Create FortiGate policies for the real LAN/transit interface pair under **Policy & Objects > Firewall Policy**. Use specific lab-network address objects, enable logging, and leave NAT off. BGP establishes to the FortiGate itself; the policy is for user traffic after routes are learned.

## Verification

```shell
get router info bgp summary
get router info bgp network
get router info bgp neighbors 10.~~.255.2 advertised-routes
get router info bgp neighbors 10.~~.255.2 received-routes
get router info routing-table bgp
get router info routing-table details 10.~~.20.10
```

Expected result:

- Neighbor state is `Established`; the summary normally shows a received-prefix count instead of an error state.
- `10.~~.10.0/24` is advertised.
- `10.~~.20.0/24` is installed as a BGP route through `10.~~.255.2`.

On Cisco:

```text
show ip bgp summary
show ip bgp
show ip route bgp
```

## Best-path basics

When several BGP paths exist, FortiGate compares BGP attributes before installing the best route. Important beginner attributes include weight, local preference, AS-path length, origin, MED, and whether the path is eBGP or iBGP. This one-peer lab deliberately avoids policy manipulation: the only valid learned path should become best.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| State remains `Idle`/`Active` | Peer IP is unreachable, neighbor is wrong, or TCP 179 cannot establish | Ping the peer, verify the connected `/30`, and inspect TCP 179 |
| Session reports bad peer AS | Local and remote AS values do not mirror | FortiGate neighbor must use `65002`; Cisco neighbor must use `65001` |
| Session is Established but local prefix is not advertised | Exact prefix is absent from the local routing table | Create/restore the connected or static route for `10.~~.10.0/24` |
| Prefix appears in BGP but not the routing table | A route from a preferred source already wins or next hop is invalid | Inspect routing-table details and BGP best-path output |
| Routers exchange routes but PCs cannot communicate | Firewall policy, endpoint gateway, or return path is wrong | Test from endpoints, inspect policy counters, and keep NAT off |
| Session resets periodically | Duplicate router ID, unstable link, or hold timer expiry | Use unique IDs and check the transit interface for loss |

Useful capture:

```shell
diagnose sniffer packet <bgp-interface> 'host 10.~~.255.2 and tcp port 179' 4 20 l
```

## Notes

- Private ASNs are appropriate for this isolated lab and must not be advertised to the public internet.
- Do not add redistribution until the basic neighbor and explicit network advertisements work.
- BGP configuration requires no FortiGuard security subscription.
