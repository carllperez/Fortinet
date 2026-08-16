# FortiGate 60F — Equal-Cost Multi-Path Routing

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Install two equal routes and observe session distribution and failure behavior |
| License | No paid FortiGuard subscription required |
| Est. time | 25–40 minutes |

## Overview

ECMP installs more than one equally preferred next hop for the same destination. FortiGate distributes sessions—not individual packets in this basic lab—across the eligible paths. A single session normally stays on one path so packet ordering is preserved.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

## Topology

```text
                       next hop 200.0.0.1 ── Router A ── 203.0.113.0/24
student PC 10.~~.10.10 ── LAN ── [ FortiGate ]
                       next hop 201.0.0.1 ── Router B ── 203.0.113.0/24
```

| Path | FortiGate interface | Gateway | Destination |
|---|---|---|---|
| A | `wan1` | `200.0.0.1` | `203.0.113.0/24` |
| B | `wan2` | `201.0.0.1` | `203.0.113.0/24` |

## Prerequisites

- Both next hops are reachable on their directly connected interfaces
- Both routers have a return route to student PC network `10.~~.10.0/24`
- The destination is reachable behind both routers
- Firewall policies allow LAN to both WAN interfaces; NAT is off for this routed private lab

## Configuration

### Step 1 — Create equal static routes

Go to **Network > Static Routes** and create two routes to `203.0.113.0/255.255.255.0`:

| Interface | Gateway | Distance | Priority |
|---|---|---:|---:|
| `wan1` | `200.0.0.1` | `10` | `0` |
| `wan2` | `201.0.0.1` | `10` | `0` |

The destination, distance, and priority must be equal for this predictable static ECMP lab. A lower priority value on one route would make it preferred instead of equal.

> **Screenshot:** Static Routes showing two active routes for `203.0.113.0/24`.

### Step 2 — Create firewall policies

Create one LAN-to-`wan1` policy and one LAN-to-`wan2` policy with the same source, destination, schedule, and services. Leave NAT off because Router A and Router B have return routes. If this were internet access using private LAN addresses, NAT would be enabled on both policies.

### Step 3 — Generate several independent sessions

Use multiple clients or multiple source/destination/port combinations. One long download is one session and will not demonstrate distribution by itself.

## Verification

```shell
get router info routing-table details 203.0.113.10
get router info routing-table all
diagnose sniffer packet wan1 'net 203.0.113.0/24' 4 20 l
diagnose sniffer packet wan2 'net 203.0.113.0/24' 4 20 l
```

Routing-table details should show both gateways as equal paths. New flows should appear on both interfaces when the configured ECMP hash has enough varied inputs.

## Link failure test

1. Start several short, repeatable connections.
2. Disconnect `wan1` or shut Router A's directly connected interface.
3. Confirm the `wan1` route leaves the active table.
4. Start new connections; they should use `wan2`.
5. Reconnect `wan1` and confirm both routes return.

Existing sessions on the failed path usually reset. ECMP does not migrate their state to a different next hop.

> Gotcha: a static route detects a physical/interface or next-hop resolution failure, not every upstream failure. If Router A's Ethernet stays up while the remote service behind it fails, plain ECMP can continue hashing sessions to the bad path.

## ECMP versus SD-WAN

| Use ECMP when | Use SD-WAN when |
|---|---|
| Paths are genuinely equivalent routed links | Internet links need active SLA probes |
| Simple session distribution is enough | Loss, latency, jitter, or application steering matters |
| Dynamic routing supplies equal paths | You need explicit failover/load-balance rules and monitoring |
| The lab is teaching route selection | The deployment needs operational visibility per ISP |

See `SDWAN.md` for the health-aware dual-internet design.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Only one route is active | Distance, priority, prefix, or mask differs | Make both routes exactly equal and verify both next hops |
| Both routes are active but one capture is quiet | Too few sessions or the ECMP hash inputs do not vary | Generate many independent sessions from several clients |
| Traffic exits both paths but replies fail | Routers lack return routes or asymmetric security devices drop replies | Add return routes and inspect both directions |
| One link fails but sessions still attempt it | Link remains electrically up and no health check detects upstream failure | Use SD-WAN or a dynamic routing protocol with failure detection |
| Internet traffic leaves with private source addresses | NAT is disabled on internet policies | Enable outgoing-interface NAT for both internet egress policies |

## Notes

- ECMP happens after longest-prefix selection; a more-specific route still wins.
- Session distribution is not a promise of exactly 50/50 bandwidth use.
- Do not clear the entire production session table for a lab test; create new sessions instead.
