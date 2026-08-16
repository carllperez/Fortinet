# FortiGate — Site-to-Site IPsec VPN with OSPF over the Tunnel

| | |
|---|---|
| Devices | FortiGate 60F / 60E (one at each site) |
| Firmware | FortiOS 7.0 |
| Method | Route-based IPsec + OSPF over the tunnel (dynamic routing) |
| License | Not required |

## Overview

Two sites connect over the WAN through an encrypted IPsec tunnel. Instead of hardcoding remote subnets, the tunnel uses `0.0.0.0/0` selectors and runs OSPF across the tunnel, so every VLAN at both sites is learned automatically — including the PC VLANs behind the switches.

> **Day 1 Cisco prerequisite:** At both sites, configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md), then complete [OSPF.md](OSPF.md). The VPN expects the FortiGate to learn the local VLANs from `COREbaba` before advertising them through the tunnel.

> Replace `~~` with the local student's monitor number and `P` with the remote student's monitor number. Working example: local monitor `61`, remote monitor `62`.

### Address plan (example 61 ↔ 62)

| Item | Local (site ~~ / 61) | Remote (site P / 62) |
|---|---|---|
| WAN (wan1) | 200.0.0.~~ / 200.0.0.61 | 200.0.0.P / 200.0.0.62 |
| FortiGate LAN (internal1) | 10.~~.~~.1 / 10.61.61.1 | 10.P.P.1 / 10.62.62.1 |
| PC VLAN (behind switch) | 10.~~.10.0/24 / 10.61.10.0/24 | 10.P.10.0/24 / 10.62.10.0/24 |
| PC address | 10.~~.10.10 / 10.61.10.10 | 10.P.10.10 / 10.62.10.10 |
| PC gateway (switch SVI) | 10.~~.10.4 / 10.61.10.4 | 10.P.10.4 / 10.62.10.4 |
| Tunnel transit /30 | 10.255.255.1 | 10.255.255.2 |

## Why this design (and why /24 selectors fail)

The PCs are **not** on the FortiGate LAN subnet — they live on VLAN 10 (`10.~~.10.0/24`) behind the core switch. A tunnel with `/24` selectors that only cover `10.~~.~~.0/24` will come Up but PCs still can't ping, because their subnet isn't inside the tunnel. Using `0.0.0.0/0` selectors + OSPF avoids this: OSPF advertises all VLANs across the tunnel.

---

## Phase 0 — Confirm the Cisco core handoff

This guide uses the supplied Day 1 switching design. At each site, `COREbaba` owns the VLAN gateways and connects to the FortiGate through a routed port:

```text
configure terminal
ip routing
interface GigabitEthernet0/1
 description ROUTED-TO-FORTIGATE-internal1
 no switchport
 ip address 10.~~.~~.4 255.255.255.0
 no shutdown
interface Vlan10
 description STUDENT-PC-WIRELESS
 ip address 10.~~.10.4 255.255.255.0
 no shutdown
router ospf 1
 router-id 10.~~.~~.4
 passive-interface default
 no passive-interface GigabitEthernet0/1
 network 10.~~.0.0 0.0.255.255 area 0
interface GigabitEthernet0/1
 ip ospf network point-to-point
end
```

The FortiGate side is `internal1` at `10.~~.~~.1/24`. It must already form a point-to-point OSPF adjacency with `COREbaba`, as documented in [OSPF.md](OSPF.md). Before building the VPN, this command must show `10.~~.10.0/24` through `10.~~.~~.4`:

```shell
get router info routing-table details 10.~~.10.10
```

Repeat the same design remotely, replacing `~~` with `P`. Do not configure `10.~~.10.4` on the FortiGate; that address belongs only to the local Cisco SVI.

## Phase 1 — Custom IPsec tunnel

`VPN > IPsec Tunnels > Create New` → Template Type: **Custom**

- Name: `VPN-to-P`
- Network > Remote Gateway: Static IP `200.0.0.P`
- **Interface: `wan1`**  ← the tunnel egresses the WAN. NOT internal1.
- Authentication: Pre-shared Key (identical both sides), IKE Version 2
- Phase 2 Selectors: Local `0.0.0.0/0`, Remote `0.0.0.0/0`

> Gotcha #1: the Phase 1 **outgoing interface must be `wan1`**. If it is set to `internal1`, the tunnel will not pass traffic — `get vpn ipsec tunnel summary` shows `selectors(total,up): 1/0` and 0 packets. Fix: `config vpn ipsec phase1-interface / edit "<name>" / set interface "wan1"`.

## Phase 2 — Tunnel interface IP

`Network > Interfaces` → expand `wan1` → edit the tunnel interface:

- IP: `10.255.255.1` (remote side uses `10.255.255.2`)
- Remote IP/Netmask: `10.255.255.2/32`
- Administrative Access: PING

## Phase 3 — Firewall policies (both directions)

`Policy & Objects > Firewall Policy` — create **two**:

| Policy | Incoming | Outgoing | Src | Dst | NAT |
|---|---|---|---|---|---|
| to-peer | **internal1** | tunnel | all | all | OFF |
| from-peer | tunnel | **internal1** | all | all | OFF |

> Gotcha #2: the outbound (`to-peer`) policy's **incoming interface must be `internal1`**, not `wan1`. LAN/switch traffic enters on internal1. If it is set to `wan1`, the FortiGate itself can still ping the peer (self-originated traffic bypasses policy) but **transit traffic from your PCs/switch is dropped**. Symptom: `execute ping` from the FortiGate works, but a ping from the switch or a PC fails. Fix: `config firewall policy / edit <id> / set srcintf "internal1"`.

## Phase 4 — OSPF over the tunnel

`Network > OSPF` (or CLI). Add the tunnel as a point-to-point OSPF interface and include the transit link.

```
config router ospf
  config ospf-interface
    edit "tunnel-ospf"
      set interface "VPN-to-P"
      set network-type point-to-point
    next
  end
  config network
    edit 4
      set prefix 10.255.255.0 255.255.255.252
      set area 0.0.0.0
    next
  end
end
```

Don't add the Cisco VLANs as FortiGate OSPF network entries — those networks are not connected FortiGate interfaces. They are learned from `COREbaba` over `internal1` and then advertised across the tunnel once both adjacencies are Full. Verify that `get router info ospf neighbor` shows the local core on `internal1` and the remote FortiGate via `10.255.255.x`, both in **Full** state.

## Phase 5 — Make the tunnel preferred over the WAN

> Gotcha #3 (the subtle one): if all sites also run OSPF over the shared WAN (`200.0.0.0/24`), each site learns the other's LANs over **both** the tunnel and the WAN. OSPF often prefers the WAN path (lower cost), so LAN-to-LAN traffic goes out `wan1` and gets dropped (there is no WAN→LAN policy). Symptom: FortiGate-to-FortiGate pings work, but PC-to-PC does not; `get router info routing-table details <remote-LAN>` shows the route `via wan1` instead of the tunnel.

Fix: give the tunnel a low OSPF cost and `wan1` a high one, on **both** FortiGates:

```
config router ospf
  config ospf-interface
    edit "tunnel-ospf"          ; the tunnel OSPF interface
      set cost 10
    next
    edit "wan1"
      set interface "wan1"
      set cost 100
    next
  end
end
```

After this, `get router info routing-table details 10.P.10.10` should show the remote PC network **via the tunnel**, not wan1.

---

## Remote site

Mirror everything at the remote site: use the `P` values on `COREbaba`, Phase 1 remote-gw `200.0.0.~~` on FortiGate `wan1`, tunnel IP `10.255.255.2` / remote `10.255.255.1`, the two firewall policies (incoming `internal1` on the outbound one), OSPF on the tunnel, the same pre-shared key, and the same Phase 5 cost change.

## PC requirements (easy to forget)

- Each PC uses address `10.~~.10.10/24` locally or `10.P.10.10/24` remotely. Its **default gateway** must be the local switch SVI (`10.~~.10.4` or `10.P.10.4`). A PC with no gateway cannot reply across the tunnel.
- Windows Firewall blocks ping from other subnets by default — allow ICMP (or disable the firewall to test) on the target PC.

## Verification (bottom-up)

Run these in order; the first failure tells you the layer:

1. `execute ping 10.255.255.2` — tunnel transit link is up.
2. `get vpn ipsec tunnel summary` — `selectors(total,up): 1/1`, packets flowing.
3. `get router info ospf neighbor` — peer via the tunnel, state Full.
4. `get router info routing-table details 10.P.10.10` — remote PC network via the **tunnel**.
5. From the FortiGate: `execute ping-options source 10.~~.~~.1` then `execute ping 10.P.P.1` — FortiGate-to-FortiGate.
6. From the local PC: `ping 10.P.10.10` — the real PC-to-PC test.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Tunnel stays Down / `selectors up: 0`, 0 packets | Phase 1 outgoing interface set to `internal1` | Set it to `wan1` |
| Tunnel Down | Pre-shared key or IKE version mismatch | Re-enter identical key / IKE v2 both sides |
| FortiGate can ping peer, but PC/switch cannot | Outbound firewall policy incoming-interface is `wan1` | Set the `to-peer` policy `srcintf` to `internal1` |
| No OSPF neighbor over the tunnel | Tunnel not up, or tunnel not added to OSPF as point-to-point on one side | Bring tunnel up; add point-to-point OSPF interface both sides |
| Tunnel neighbor is Full, but the local PC VLAN is absent | FortiGate and `COREbaba` are not OSPF neighbors, or VLAN 10 SVI is down | Complete Phase 0 and verify `10.~~.10.0/24` is learned through `10.~~.~~.4` |
| FortiGate-to-FortiGate works, PC-to-PC fails; remote LAN routes `via wan1` | WAN OSPF preferred over the tunnel | Phase 5 cost change on both units (tunnel 10, wan1 100) |
| PC reaches remote FortiGate but not the remote PC | Remote PC has the wrong address/gateway or blocks ICMP | Set remote PC to `10.P.10.10/24`, gateway `10.P.10.4`, and allow ICMP |

## Notes

- Phase 2 selectors with no `src-subnet`/`dst-subnet` default to `0.0.0.0/0` — the encryption domain is wide open, so routing (not selectors) decides what crosses.
- `execute ping` from the FortiGate is self-originated (local-out) and skips firewall policy — it is not a valid test of transit traffic. Always confirm with a ping from a PC or the switch.
- Serial console at 9600 baud can garble fast multi-line paste; enter config one line at a time and verify the prompt changes (e.g. `(ospf)`).
