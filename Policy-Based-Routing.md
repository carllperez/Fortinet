# FortiGate 60F — Policy-Based Routing on Dual WAN

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Steer selected sessions to a specific WAN before normal route lookup |
| License | No paid FortiGuard subscription required |
| Est. time | 25–35 minutes |

## Overview

Normal routing selects a path mainly from the destination address. A policy route can additionally match incoming interface, source, destination, protocol, or destination port and force a usable gateway/interface. FortiGate checks policy routes from top to bottom; if none matches, it uses the normal routing table.

This lab sends guest web traffic through `wan2` while all other PC traffic follows `wan1`.

> If the WANs are already members of `virtual-wan-link`, complete `SDWAN.md` and use SD-WAN rules for health-aware steering. Do not build this lab by pulling live interfaces out of SD-WAN.
>
> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

## Network overview

| Item | Value |
|---|---|
| PC source | `10.~~.10.0/24` on `PC-VLAN10`; student PC `10.~~.10.10` |
| Guest source | `10.~~.20.0/24` on `GUEST-VLAN20` |
| `wan1` | `200.0.0.~~/24`, gateway `200.0.0.1` |
| `wan2` | `201.0.0.~~/24`, gateway `201.0.0.1` |
| Normal default | `wan1` |
| Policy-routed traffic | Guest TCP 80/443 through `wan2` |

## Prerequisites

- Both WAN interfaces are independently addressed and operational
- Both gateways are directly reachable
- A default route exists through `wan1`
- A route through `wan2` exists and is active enough to resolve `201.0.0.1`
- Matching firewall policies and NAT exist for both possible egress interfaces

## Configuration

### Step 1 — Keep both WAN paths routable

Under **Network > Static Routes**, keep the primary default through `wan1`. Add a second default through `wan2` with a higher distance, such as `20`. The backup may not be the normal best route, but it makes the `wan2` gateway resolvable for the policy route.

### Step 2 — Create the source-based web policy route

1. Go to **Network > Policy Routes**.
2. Click **Create New > Policy Route**.
3. Configure:

| Field | Value |
|---|---|
| Incoming interface | `GUEST-VLAN20` |
| Source address | `10.~~.20.0/255.255.255.0` |
| Destination address | `0.0.0.0/0.0.0.0` |
| Protocol | TCP (`6`) |
| Destination ports | `80–80` for the first rule |
| Outgoing interface | `wan2` |
| Gateway address | `201.0.0.1` |

4. Create a second adjacent rule for destination port `443`.
5. Place both above broader policy routes.

> **Screenshot:** Network > Policy Routes showing the guest HTTP and HTTPS rules above any general rule.

This demonstrates source-, protocol-, and service-based matching together. A destination-based example would replace `0.0.0.0/0` with a specific remote subnet.

### Step 3 — Create the firewall policies

Under **Policy & Objects > Firewall Policy**, create or confirm:

| Name | Incoming | Outgoing | Source | Destination | NAT |
|---|---|---|---|---|---|
| `Guest-to-wan2` | `GUEST-VLAN20` | `wan2` | `GUEST-NET` | `all` | On |
| `PC-to-wan1` | `PC-VLAN10` | `wan1` | `PC-NET` | `all` | On |

Routing and firewall policy are separate decisions. If the policy route selects `wan2` but only a `wan1` policy exists, the session is denied.

## Verification

```shell
diagnose firewall proute list
get router info routing-table all
diagnose sniffer packet wan2 'tcp port 80 or tcp port 443' 4 20 l
```

From a guest client, open a new HTTP/HTTPS session and observe it on `wan2`. Use a new session after each change; an established session keeps its original route.

For flow debugging, filter narrowly:

```shell
diagnose debug reset
diagnose debug flow filter saddr 10.~~.20.100
diagnose debug flow trace start 20
diagnose debug enable
```

Stop with `diagnose debug disable` and `diagnose debug reset`. The trace should show a policy-route match and `wan2` as egress.

## Interaction with SD-WAN

PBR is appropriate for a small, deterministic exception. It does not perform SLA probes and can continue choosing a gateway whose upstream service has failed while the Ethernet link remains up. SD-WAN rules are the better tool for health-aware failover, quality selection, and load distribution. Avoid overlapping a policy route and an SD-WAN rule unless the resulting order has been tested deliberately.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Rule never matches | Wrong incoming interface, mask, protocol, port, or rule order | Compare live packet fields and move the most specific rule upward |
| PBR selects `wan2` but traffic is denied | Firewall policy does not use `wan2` as outgoing interface | Add the correct interface-pair policy with NAT |
| PBR is skipped | Gateway is not reachable through the selected interface | Keep a valid route that resolves the gateway on `wan2` |
| HTTPS uses `wan2` but DNS uses `wan1` | Only TCP 443 was matched | This is expected; add a deliberate DNS rule only if the design requires it |
| Existing browser connection stays on old WAN | FortiGate sessions retain their chosen path | Close the connection or clear only the specific test session |
| Traffic does not fail over when ISP upstream dies | Policy routes have no SLA health check | Use SD-WAN for automatic health-based behavior |

## Notes

- The list position is the policy-route priority: first complete match wins.
- A policy route still requires a valid route to its gateway.
- FortiGate local-out traffic generally does not validate an incoming-interface PBR rule; test from a client.
