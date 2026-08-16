# FortiGate 60F — Remote Access IPsec VPN (FortiClient Dial-up)

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Remote access to the home LAN with FortiClient over IPsec (dial-up) |
| License | Not required |
| Est. time | 25–40 minutes |

## Overview

This guide configures a **dial-up (remote access) IPsec VPN** on a FortiGate 60F so a remote device running FortiClient can reach the home LAN over an encrypted IKE/ESP tunnel. Clients get an IP from a mode-config pool, authenticate with a username/password, and use split tunneling so they keep their own internet while connected.

Dial-up means the FortiGate does **not** know the client's public IP in advance. The tunnel is `type dynamic`: any client that presents the correct pre-shared key and passes user authentication is allowed to connect and is handed an address from a pool.

> Replace `~~` with your site number throughout. Example: site 61 gives LAN `10.61.61.0/24` and WAN `200.0.0.61`.

## How this differs from `SSLVPN.md`

Both guides give a remote user access to the same LAN with FortiClient and split tunneling. The difference is the transport and how the client is admitted.

| | SSL VPN (`SSLVPN.md`) | Remote Access IPsec (this guide) |
|---|---|---|
| Transport | TLS over **TCP 10443** | IKE **UDP 500**, NAT-T **UDP 4500**, **ESP (IP proto 50)** |
| Client | FortiClient tunnel **or** clientless web portal | FortiClient only (no browser portal) |
| Client IP assignment | SSL-VPN address pool (`SSLVPN_TUNNEL_ADDR1`) | IKE **mode-config** pool |
| User auth | User group on the portal mapping | **XAuth** (IKEv1) against a user group, on top of the PSK |
| Tunnel auth | Server certificate | **Pre-shared key** |
| Firewall traversal | Easy — one outbound TCP port | Needs UDP 500/4500 **and** ESP allowed to reach `wan1` |
| Performance on 60F | More CPU-bound (TLS) | Hardware-accelerated IPsec, usually higher throughput |

Practical takeaway: SSL VPN is easier to get through hotel/coffee-shop firewalls (single TCP port). IPsec is faster and is the direction Fortinet is steering remote access in newer releases, but it needs the ISP/upstream path to pass UDP 4500 and ESP. If a client can open `https://200.0.0.~~:10443` but the IPsec tunnel won't build from the same spot, suspect ESP/UDP-4500 filtering upstream.

## Prerequisites

- FortiGate 60F reachable on the GUI (`https://10.~~.~~.1`) with admin access to GUI and CLI.
- `wan1` has the public/edge address `200.0.0.~~` and is reachable from the client.
- FortiClient installed on the remote device. The free VPN-only mode is enough — no paid license or EMS is required for a manual IPsec connection.
- A client pool subnet that does **not** overlap the LAN. This guide uses `10.~~.99.0/24`.

## Network overview

| Item | Value |
|---|---|
| LAN interface | internal1 |
| LAN subnet (reachable over VPN) | 10.~~.~~.0/24 |
| WAN interface | wan1 |
| WAN IP (VPN endpoint) | 200.0.0.~~ |
| Tunnel name | RA-FortiClient |
| Client mode-config pool | 10.~~.99.1 – 10.~~.99.10 (/24) |
| User group | IPsecVPN-Users |
| IKE version (wizard default) | IKEv1 aggressive + XAuth |

## Topology

```text
Remote device (FortiClient)
        |
   Internet / WAN
        |
   wan1  200.0.0.~~
        |
   FortiGate 60F   <-- IKE/ESP tunnel, mode-config pool 10.~~.99.0/24
        |
 internal1  10.~~.~~.1
        |
   Home LAN 10.~~.~~.0/24
```

## Configuration

The `User & Authentication` steps are the same idea as the SSL VPN guide: a local user inside a user group. If you already made `SSLVPN-Users` you can reuse a user, but create a **separate group** for IPsec so the two VPNs stay independent.

### Step 1 — Create the VPN user

1. Go to **User & Authentication > User Definition**.
2. Click **Create New**.
3. Choose **Local User**, then **Next**.
4. Set a **Username** and **Password**, click through the remaining screens, and **Submit**.

> **Screenshot:** User & Authentication > User Definition wizard with Local User selected.

### Step 2 — Create the user group

1. Go to **User & Authentication > User Groups**.
2. Click **Create New**.
3. Name: `IPsecVPN-Users`.
4. Type: **Firewall**.
5. Under **Members**, add the user from Step 1.
6. Click **OK**.

This group is what XAuth checks the username/password against at connect time.

> **Screenshot:** User & Authentication > User Groups showing IPsecVPN-Users with the member added.

### Step 3 — Create the address objects

Two objects: the LAN the client will reach, and the client pool used by the firewall policy.

1. Go to **Policy & Objects > Addresses > Create New > Address**.
2. First object — the LAN:
   - Name: `LAN-~~`
   - Type: **Subnet**
   - Subnet/IP Range: `10.~~.~~.0/24`
   - Interface: **any**
3. Click **OK**, then create the second object — the VPN pool:
   - Name: `VPN-Pool-~~`
   - Type: **Subnet**
   - Subnet/IP Range: `10.~~.99.0/24`
   - Interface: **any**
4. Click **OK**.

> Set Interface to `any` on both. Binding an address to internal1 hides it from some selection menus (same behavior noted in `SSLVPN.md`).

> **Screenshot:** Policy & Objects > Addresses listing LAN-~~ and VPN-Pool-~~.

### Step 4 — Run the Remote Access wizard

1. Go to **VPN > IPsec Wizard** (or **VPN > IPsec Tunnels > Create New**).
2. **Name:** `RA-FortiClient`.
3. **Template Type:** **Remote Access**.
4. **Remote Device Type:** **FortiClient**.
5. Click **Next**.
6. **Incoming Interface:** `wan1`.
7. **Authentication Method:** **Pre-shared Key**.
8. **Pre-shared Key:** enter a strong key (you will type the same key into FortiClient).
9. **User Group:** `IPsecVPN-Users`.
10. Click **Next**.
11. **Local Interface:** `internal1`.
12. **Local Address:** `LAN-~~`.
13. **Client Address Range:** `10.~~.99.1 - 10.~~.99.10`.
14. **Subnet Mask:** `255.255.255.0`.
15. **DNS Server:** leave as system DNS, or specify `10.~~.~~.1` if you want name resolution through the FortiGate.
16. **Enable IPv4 Split Tunnel:** **on**, and set **Accessible Networks** to `LAN-~~`.
17. Client options (Save Password / Auto Connect / Always Up) are optional; leave them as you prefer.
18. Click **Create** / **Next** to finish.

The wizard builds Phase 1, Phase 2, and the tunnel-to-internal1 firewall policy for you. For a FortiClient remote-access template, FortiOS 7.0 generates an **IKEv1 aggressive-mode** tunnel with **XAuth** for user login and **mode-config** for IP assignment.

> Aggressive mode is used because, with a pre-shared key and an unknown client IP, main mode cannot tell dial-up peers apart before the key is verified. Aggressive mode lets the FortiGate identify the dial-up group first. This is expected for PSK dial-up; it is not an error.

> **Screenshot:** VPN > IPsec Wizard, Remote Access template with FortiClient selected.

> **Screenshot:** Wizard policy & routing page showing Local Address LAN-~~, client range 10.~~.99.1–10, and Split Tunnel enabled.

### Step 5 — Review the generated tunnel (CLI)

Confirm what the wizard produced. This is also the fastest way to see the settings you will mirror in FortiClient.

```bash
show vpn ipsec phase1-interface RA-FortiClient
show vpn ipsec phase2-interface RA-FortiClient
```

You should see roughly this (proposal strings vary by build — keep them matched on both ends):

```bash
config vpn ipsec phase1-interface
    edit "RA-FortiClient"
        set type dynamic
        set interface "wan1"
        set ike-version 1
        set mode aggressive
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal aes256-sha256 aes128-sha256
        set dhgrp 14
        set xauthtype auto
        set authusrgrp "IPsecVPN-Users"
        set ipv4-start-ip 10.~~.99.1
        set ipv4-end-ip 10.~~.99.10
        set ipv4-netmask 255.255.255.0
        set ipv4-split-include "LAN-~~"
        set ipv4-dns-server1 10.~~.~~.1
        set psksecret ENC <hidden>
    next
end
```

Key fields and why they matter:

- `type dynamic` — this is a dial-up tunnel; the remote gateway IP is not fixed.
- `net-device disable` — all dial-up clients share this one tunnel interface. Leave it disabled for a simple single-phase1 dial-up. `enable` creates a per-client virtual interface and then needs dynamic routing, which is overkill here.
- `mode-cfg enable` + `ipv4-start-ip`/`ipv4-end-ip` — hands the client an address from the pool. With mode-config, the FortiGate automatically installs a route to each assigned client IP through the tunnel, so no static route is needed for the return path **to the client**.
- `xauthtype auto` + `authusrgrp` — the username/password check (XAuth) layered on top of the PSK.
- `ipv4-split-include "LAN-~~"` — the only subnet pushed to the client as a tunnel route. Everything else stays on the client's own internet (split tunnel).

If you want **IKEv2** instead, it is supported but is not what the FortiClient wizard builds. IKEv2 replaces XAuth with EAP for user login (`set ike-version 2`, `set eap enable`, `set eap-identity send-request`, keep `set authusrgrp`), and you would build it from the **Custom** template. Verify FortiClient's IKEv2/EAP settings match before relying on it. If you are not sure, stay on the wizard's IKEv1 build — it is the well-trodden path on FortiOS 7.0.

### Step 6 — Check the firewall policy

The wizard creates a policy from the tunnel to `internal1`. Confirm it under **Policy & Objects > Firewall Policy**:

| Field | Value |
|---|---|
| Name | RA-FortiClient (or wizard-generated) |
| Incoming Interface | RA-FortiClient (tunnel) |
| Outgoing Interface | internal1 |
| Source | VPN-Pool-~~ (and optionally the IPsecVPN-Users group) |
| Destination | LAN-~~ |
| Schedule | always |
| Service | ALL |
| Action | ACCEPT |
| NAT | **OFF** |

CLI equivalent:

```bash
config firewall policy
    edit 0
        set name "RA-FortiClient"
        set srcintf "RA-FortiClient"
        set dstintf "internal1"
        set srcaddr "VPN-Pool-~~"
        set dstaddr "LAN-~~"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

> NAT is **OFF** here on purpose. This is routed private-to-private traffic. With NAT off, LAN hosts see the real client-pool IP (`10.~~.99.x`) and send replies back to their default gateway, the FortiGate, which routes them into the tunnel. Turning NAT on would hide every client behind `10.~~.~~.1` — only do that if you cannot fix return routing (see the gotcha below).

> Gotcha: return routing when PCs sit behind a switch.
> Symptom: the VPN client can ping the FortiGate LAN IP `10.~~.~~.1`, but cannot reach PCs on a VLAN behind the core switch (e.g. `10.~~.1.0/24`).
> Cause: those PCs (or the switch) have no route back to the VPN pool `10.~~.99.0/24`; their default gateway is the switch SVI, not the FortiGate.
> Fix: add a route on the L3 switch for `10.~~.99.0/24` pointing at the FortiGate (`10.~~.1.4` → `10.~~.~~.1`), **or** enable NAT on this policy so clients are source-NATed to `10.~~.~~.1`. NAT is the quick homelab fix — the same trick used in `SSLVPN.md`. Also add `10.~~.1.0/24` to the split-include address if you want that VLAN reachable.

Client-initiated sessions only need this one policy; the reply traffic is allowed by the session table. Add a reverse `internal1 → tunnel` policy only if LAN hosts must start connections **to** the clients.

> **Screenshot:** Policy & Objects > Firewall Policy showing the RA-FortiClient tunnel-to-internal1 policy with NAT disabled.

### Step 7 — Configure FortiClient and connect

1. In FortiClient open **Remote Access** and add a new **VPN** connection.

| Field | Value |
|---|---|
| VPN Type | IPsec VPN |
| Remote Gateway | 200.0.0.~~ |
| Authentication Method | Pre-Shared Key |
| Pre-shared Key | the key from Step 4 |

2. If FortiClient exposes Phase 1/Phase 2 detail, set IKE version to match (IKEv1, aggressive), and let mode-config assign the IP/DNS.
3. Save, then **Connect**.
4. Enter the **username and password** (the XAuth prompt) from Step 1.

> The client must reach `200.0.0.~~` on **UDP 500 and UDP 4500**, and ESP must not be blocked upstream. Most clients are behind NAT, so the tunnel uses NAT-T on UDP 4500.

> **Screenshot:** FortiClient IPsec VPN connection settings (gateway 200.0.0.~~, PSK auth).

## Verification

### GUI

1. **VPN > IPsec Tunnels** — the `RA-FortiClient` tunnel shows **Up** with a connected-client count of 1 while FortiClient is connected.
2. **Dashboard > Network > IPsec** widget also shows the active dial-up tunnel.
3. In FortiClient, the status shows **Connected** with an assigned IP in `10.~~.99.x`.

> **Screenshot:** VPN > IPsec Tunnels with RA-FortiClient Up and one connected client.

### CLI

```bash
get vpn ipsec tunnel summary
diagnose vpn tunnel dialup-list
diagnose vpn ike gateway list
```

What to expect:

- `get vpn ipsec tunnel summary` — lists `RA-FortiClient`; once a client connects, `selectors(total,up)` shows an up selector and the rx/tx byte counters increment.
- `diagnose vpn tunnel dialup-list` — one row per connected client, showing the client's public IP, the XAuth **username**, and the **assigned IP** from the pool. An empty list means no client is currently connected (or auth is failing before mode-config).
- `diagnose vpn ike gateway list` — the IKE SA is `established`, with the peer set to the client's public address.

Then prove data path:

1. With the VPN connected, from the client ping `10.~~.~~.1`. It should reply.
2. Ping or open a real LAN host inside `10.~~.~~.0/24`.
3. Confirm the client still browses the internet — that proves split tunneling is active.

Packet capture on the FortiGate while the client pings a LAN host (use the client's assigned IP):

```bash
diagnose sniffer packet any 'host 10.~~.99.1' 4 0 l
```

You should see the echo request arrive on the tunnel interface and leave on `internal1`, with a reply coming back — if you see the request go out internal1 but never a reply, it is a return-routing problem (see the Step 6 gotcha).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| FortiClient stuck at "Connecting", Phase 1 never completes | PSK mismatch, or IKE version/mode mismatch | Re-enter the identical pre-shared key; match IKEv1 aggressive on both sides; watch negotiation with `diagnose debug application ike -1` then `diagnose debug enable` |
| Tunnel builds, then "XAuth failed" / wrong username or password | User not in the group set as `authusrgrp`, or bad credentials | Add the user to `IPsecVPN-Users`; confirm the group is bound as XAuth group; `diagnose vpn tunnel dialup-list` stays empty when XAuth fails |
| Client connects but gets no IP address | mode-config pool missing or exhausted | Confirm `mode-cfg enable` and `ipv4-start-ip`/`ipv4-end-ip`; widen the range if all leases are in use |
| Client has an IP but cannot reach the LAN at all | Missing or wrong tunnel→internal1 firewall policy | Create/verify the policy: srcintf the tunnel, dstintf internal1, dstaddr `LAN-~~`, action accept |
| Client pings `10.~~.~~.1` but not PCs behind the switch | PCs/switch have no route back to the VPN pool | Add a route for `10.~~.99.0/24` on the L3 switch via the FortiGate, or enable NAT on the policy; add the PC VLAN to the split-include |
| Client's internet dies while connected | Split tunnel not applied (full tunnel) | Set `ipv4-split-include` to `LAN-~~`; reconnect FortiClient (routes load at connect time) |
| Works on the LAN/test but not from the real internet | UDP 500/4500 or ESP blocked upstream, or `wan1` unreachable | Confirm `200.0.0.~~` is reachable; ensure NAT-T (UDP 4500) and ESP pass through the upstream/ISP; CGNAT can block ESP inbound |
| Sites open by IP but not by name | Pushed DNS server is unreachable | Set a reachable `ipv4-dns-server1` (e.g. `10.~~.~~.1`) or leave DNS to the client system, then reconnect |
| A second client knocks the first offline | Two clients logging in with the same user account | Give each client its own user; `net-device disable` on a shared dial-up phase1 is normal and supports multiple distinct clients |
| `execute ping` from the FortiGate works but the client's traffic fails | Self-originated traffic bypasses policy; it is not a transit test | Test from the client and check the transit policy and return route, not the FortiGate's own ping |

## Notes

- Ports and protocols: IKE **UDP 500**, NAT-T **UDP 4500**, and **ESP (IP protocol 50)**. IPsec is not a single TCP port like SSL VPN — the upstream path must pass all three (4500 + ESP in practice, since clients are usually behind NAT).
- NAT direction: **OFF** for reaching the private LAN (routed private-to-private), which preserves the client-pool source IP and relies on correct return routing. Turn NAT **ON** only as a homelab shortcut when you cannot add a return route on the LAN side.
- mode-config auto-adds the route to each connected client's assigned IP through the tunnel, so you do not add a static route for the pool. You only worry about routing on the **LAN side** (hosts/switch getting replies back to the FortiGate).
- The FortiGate 60F accelerates IPsec in hardware, so a dial-up IPsec tunnel usually gives higher throughput than SSL VPN, which is more CPU-bound. Both are free.
- Keep the client pool (`10.~~.99.0/24`) off the LAN subnet. Overlapping the pool with `10.~~.~~.0/24` breaks routing and causes intermittent "reaches some hosts, not others" behavior.
- Cross-references: if the LAN you need to reach is a VLAN behind a switch, set it up with `VLAN.md` first, add that subnet to the split-include, and handle return routing (the same routing concern covered in `Site-to-Site-VPN.md`). For a browser-friendly, single-TCP-port alternative to this tunnel, see `SSLVPN.md`.
- In FortiOS releases after 7.0, Fortinet is moving remote access toward IPsec as SSL VPN tunnel mode is phased out on some models. This guide targets FortiOS 7.0.x, where both are fully supported; do not mix newer-release GUI steps into this configuration.
