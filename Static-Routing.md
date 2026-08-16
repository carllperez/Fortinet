# FortiGate 60F — Static Routing Fundamentals

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Route a lab LAN, a downstream subnet, and a default path |
| License | No paid FortiGuard subscription required |
| Est. time | 20–30 minutes |

## Overview

Connected routes appear automatically when an addressed interface is up. Static routes are administrator-created instructions for destinations behind another router. A default route (`0.0.0.0/0`) is the least-specific route and is used only when no more-specific route matches.

> **Day 1 Cisco prerequisite:** When using the supplied Cisco topology, configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) first. This lab uses `COREbaba` as the downstream router at `10.~~.~~.4`.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives student PC `10.61.10.10`.

## Topology

```text
Internet router 200.0.0.1
          |
wan1 200.0.0.~~/24
    [ FortiGate ] internal1 10.~~.~~.1/24
          |
  Cisco COREbaba 10.~~.~~.4
          |
  PC network 10.~~.10.0/24
  gateway 10.~~.10.4 ── student PC 10.~~.10.10
```

| Route | Gateway | Interface | Purpose |
|---|---|---|---|
| `10.~~.~~.0/24` | Connected | `internal1` | Local LAN |
| `10.~~.10.0/24` | `10.~~.~~.4` | `internal1` | Student PC network behind `COREbaba` |
| `0.0.0.0/0` | `200.0.0.1` | `wan1` | All other destinations |

## Configuration

### Step 1 — Confirm connected routes

Go to **Network > Interfaces** and confirm both addressed interfaces are up. Then go to **Dashboard > Network**, expand the Routing widget, or run:

```shell
get router info routing-table connected
```

If an interface is administratively up but the physical link is down, its connected route may not be usable.

### Step 2 — Add the downstream static route

1. Go to **Network > Static Routes**.
2. Click **Create New**.
3. Set Destination to `10.~~.10.0/255.255.255.0`.
4. Set Gateway to `10.~~.~~.4`.
5. Set Interface to `internal1`.
6. Leave Administrative Distance at `10` and Priority at `0` for this single-path lab.
7. Save the route.

> **Screenshot:** Static route to student PC network `10.~~.10.0/24` through Cisco `COREbaba`.

The gateway must be reachable on the selected outgoing interface. A next hop outside `10.~~.~~.0/24` will not resolve without another route.

Cisco `COREbaba` uses routed `Gi0/1` address `10.~~.~~.4/24` toward the FortiGate and VLAN 10 SVI `10.~~.10.4/24` toward the PC. Enable `ip routing` and give it default route `0.0.0.0/0` through `10.~~.~~.1` so the student PC has a return path through the FortiGate.

### Step 3 — Add the default route

Create another route:

| Field | Value |
|---|---|
| Destination | `0.0.0.0/0.0.0.0` |
| Gateway | `200.0.0.1` |
| Interface | `wan1` |
| Distance | `10` |
| Priority | `0` |

Distance compares routes learned by different sources or otherwise competing routes; lower is preferred. Priority is a FortiGate tie-breaker among otherwise equal static routes; lower is preferred. Neither value overrides longest-prefix matching.

### Step 4 — Add transit firewall policies

Routing chooses an egress interface; it does not permit traffic. Create only the policies the lab needs. For LAN-to-downstream traffic use `internal1` as both incoming and outgoing interface only if both networks really share that same FortiGate interface; otherwise select the actual interface pair. Leave NAT off and make sure the downstream router has a return route to the source network.

For LAN-to-internet traffic, use `internal1` to `wan1` and enable NAT using the outgoing-interface address.

## How route selection works

FortiGate first chooses the longest matching prefix. For destination `10.~~.10.10`, the `/24` route wins over the default `/0` even if the default route has a lower administrative distance. Distance and priority matter only after the best prefix length has been identified.

## Verification

```shell
get router info routing-table all
get router info routing-table details 10.~~.10.10
get router info routing-table details 8.8.8.8
execute ping-options source 10.~~.~~.1
execute ping 10.~~.10.10
```

Expected results:

- `C` marks the connected LAN and WAN networks.
- `S` marks the downstream and default static routes.
- Details for `10.~~.10.10` show `10.~~.~~.4` on `internal1`.
- Details for `8.8.8.8` show `200.0.0.1` on `wan1`.

Also test from the student PC at `10.~~.10.10/24`, gateway `10.~~.10.4`. A FortiGate ping is local-out traffic and does not validate a transit firewall policy.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Static route is configured but absent from the active table | Interface is down or gateway is not resolvable through it | Restore link and correct the gateway/interface pair |
| FortiGate can ping downstream but the PC cannot | Missing transit policy or downstream return route | Check policy counters and add a route back to the PC subnet |
| Traffic follows the default route instead of the downstream route | Destination mask is wrong or the specific route is inactive | Correct the `/24` and verify next-hop reachability |
| Preferred backup route is selected unexpectedly | Distance or static-route priority is lower than intended | Compare equal-prefix routes and use lower values only on the preferred path |
| Internet works from FortiGate but not clients | LAN-to-WAN policy or source NAT is missing | Enable the correct policy and outgoing-interface NAT |
| Ping sourced from the wrong address gives misleading results | FortiGate selected an interface address automatically | Set `execute ping-options source`, test, then run `execute ping-options reset` |

## Notes

- Do not add NAT to private routed paths merely to hide a missing return route; add the return route when you control the other router.
- Use this static route only when OSPF from `OSPF.md` is not providing the same prefix; avoid keeping unintended competing routes.
- Use `Policy-Based-Routing.md` only when source, service, or another policy criterion must override ordinary destination routing.
