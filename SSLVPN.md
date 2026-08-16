# FortiGate 60F — SSL VPN Installation and Setup

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0 |
| Purpose | FortiClient tunnel access to the main LAN and student PC VLAN |
| License | Not required |
| Est. time | 20–30 minutes |

## Overview

This guide configures an SSL VPN on a FortiGate 60F so a remote device running FortiClient in tunnel mode can securely reach the main LAN and the student PC VLAN. Split tunneling is used so the client keeps its own internet while connected.

> **Day 1 Cisco prerequisite:** Configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md), then confirm the FortiGate route to the student PC VLAN with [OSPF.md](OSPF.md) or the documented static route.

> This is a **FortiClient tunnel-mode** guide. Opening the SSL VPN URL in a browser is useful for checking that the service is reachable, but browser-based access to internal resources requires a separately configured web-mode portal and bookmarks; that is outside this guide.

> Replace `~~` with the student's monitor number throughout. Example: monitor 61 gives student PC `10.61.10.10`, LAN `10.61.61.0/24`, and WAN `200.0.0.61`.
>
> Screenshots are hosted on GitHub (user-attachments) and embedded inline below — no `images/` folder needed. Committing this single file is enough.

## Prerequisites

- FortiGate 60F reachable on the GUI (`https://10.~~.~~.1`)
- Admin username and password
- FortiClient installed on the phone or laptop that will connect
- The client device on a network that can reach the FortiGate WAN (`200.0.0.~~`)

## Network overview

| Item | Value |
|---|---|
| LAN interface | internal1 |
| LAN subnet | 10.~~.~~.0/24 |
| WAN interface | wan1 |
| WAN IP | 200.0.0.~~ |
| SSL VPN port | 10443 |
| Tunnel client pool | SSLVPN_TUNNEL_ADDR1 (default) |
| Student PC VLAN | 10.~~.10.0/24 |
| Student PC | 10.~~.10.10/24 |
| Student PC gateway | 10.~~.10.4 |

The student PC subnet is separate from the FortiGate LAN subnet. This guide includes both networks in the split tunnel. It follows the same topology as the Site-to-Site VPN lab: the core switch owns the PC gateway `10.~~.10.4`, and the FortiGate reaches `10.~~.10.0/24` through `internal1`. Confirm that the FortiGate already has an OSPF or static route to the PC VLAN before continuing.

### Cisco switching prerequisite

In the supplied Day 1 topology, Cisco `COREbaba` must have `ip routing`, VLAN 10 SVI `10.~~.10.4/24`, and routed `Gi0/1` address `10.~~.~~.4/24` toward FortiGate `internal1` (`10.~~.~~.1/24`). Follow [VLAN.md](VLAN.md) for switching and [OSPF.md](OSPF.md) for route exchange.

Confirm the FortiGate route before configuring SSL VPN:

```shell
get router info routing-table details 10.~~.10.10
```

The next hop should be `10.~~.~~.4` on `internal1` (directly or through OSPF). Do not create a FortiGate `PC-VLAN10` interface or assign `10.~~.10.4` to the FortiGate in this design.

## Configuration

### Step 1 — Create the VPN user

1. Go to User & Authentication > User Definition.
2. Click Create New.
3. Choose Local User, then Next.
4. Set a Username and Password.
5. Click through the remaining screens and Submit.

![Step 1a — User creation wizard: choose Local User](https://github.com/user-attachments/assets/b8925eae-f3ef-4818-9eaf-6095ee109537)

![Step 1b — Login Credentials: set username and password](https://github.com/user-attachments/assets/9649074a-1aec-4812-83f6-6510ea0b9305)

![Step 1c — Contact Info (optional, click through)](https://github.com/user-attachments/assets/21a11b87-5024-496e-b226-66db815da2c5)

![Step 1d — Extra Info (optional, click through)](https://github.com/user-attachments/assets/93beb0ce-c5b3-45a6-9a27-8ea2c7529cf5)

### Step 2 — Create a user group

1. Go to User & Authentication > User Groups.
2. Click Create New.
3. Name: `SSLVPN-Users`.
4. Type: Firewall.
5. Under Members, add the user from Step 1.
6. Click OK.

![Step 2 — New User Group SSLVPN-Users (Firewall) with the user added](https://github.com/user-attachments/assets/41d27125-e488-4af2-861e-465727af9fce)

### Step 3 — Create the protected-network address objects

1. Go to Policy & Objects > Addresses.
2. Click Create New > Address.
3. Name: `LAN-~~`.
4. Type: Subnet.
5. Subnet/IP Range: `10.~~.~~.0/24`.
6. Interface: any.
7. Click OK.

Create a second address object:

1. Name: `PC-VLAN-~~`.
2. Type: Subnet.
3. Subnet/IP Range: `10.~~.10.0/24`.
4. Interface: any.
5. Click OK.

Then create an address group:

1. Go to **Policy & Objects > Addresses** and click **Create New > Address Group**.
2. Name: `SSLVPN-Destinations`.
3. Add `LAN-~~` and `PC-VLAN-~~` as members.
4. Click OK.

> Set both address objects to Interface `any`. Binding them to an interface can hide them from required selection menus. The firewall policy still controls the actual outgoing interface.

![Step 3a — Policy & Objects > Addresses: Create New > Address](https://github.com/user-attachments/assets/3ebdc109-5b44-4492-b2a8-346a59424cf3)

![Step 3b — Edit Address: LAN subnet object, Interface set to any](https://github.com/user-attachments/assets/8b065ba8-55b6-4de7-b991-3a71b9c8aaac)

### Step 4 — Configure the SSL VPN portal

1. Go to VPN > SSL-VPN Portals.
2. Edit the `full-access` portal.
3. Enable Tunnel Mode.
4. For Split Tunneling, choose **Enabled Based on Policy Destinations**. The dropdown has three options: Disabled, Enabled Based on Policy Destinations, and Enabled for Trusted Destinations.
5. Click OK.

> Enabled Based on Policy Destinations builds the tunnel routes automatically from the destination of the SSL-VPN firewall policy (Step 6). With that policy's destination set to `SSLVPN-Destinations`, both `10.~~.~~.0/24` and `10.~~.10.0/24` go through the tunnel while the client keeps its own internet.
> This mode requires the policy destination to be a specific address or address group, not `all`, or it fails with "could not enable split tunneling as policy has all." Create the address objects and group in Step 3, then use the group in Step 6.
> Note: "Enabled for Trusted Destinations" (manual Routing Address Override) is meant to do the same thing, but it did not reliably push the split routes to the mobile FortiClient in this setup — the phone full-tunneled and lost internet. Policy Destinations worked.

![Step 4 — SSL-VPN Portal full-access: Tunnel Mode on, Enabled Based on Policy Destination](https://github.com/user-attachments/assets/979334b6-a882-47b2-90b4-407e2d3e1915)

### Step 5 — Configure SSL VPN settings

1. Go to VPN > SSL-VPN Settings.
2. Listen on Interface: wan1.
3. Listen on Port: `10443` (443 is the admin GUI, keep them separate).
4. Server Certificate: Fortinet_Factory (self-signed; the browser will warn, which is expected).
5. DNS Server: set to **Same as client system DNS** so the FortiGate does not push an unreachable resolver. (Blank can still push the FortiGate's own system DNS, which clients often cannot reach — that shows up as "Primary DNS is unreachable" and no internet.)
6. Under Authentication/Portal Mapping, click Create New and map `SSLVPN-Users` to the `full-access` portal.
7. Set the **All Other Users/Groups** row to `no-access`. It cannot be blank, and `no-access` prevents unmatched users from receiving an unintended portal.
8. Click Apply.

![Step 5a — SSL-VPN Settings: listen on wan1, port 10443, Fortinet_Factory certificate](https://github.com/user-attachments/assets/5b6b5d71-230d-491c-b0a9-2f7d2d619b5f)

![Step 5b — SSL-VPN Settings: DNS and portal mapping from the original lab](https://github.com/user-attachments/assets/3125e5c0-d1ed-48fe-98fc-2c8495893c8d)

> The original screenshot shows `web-access` for unmatched users. For the final configuration, select `no-access` as instructed above.

### Step 6 — Create the firewall policy

1. Go to Policy & Objects > Firewall Policy.
2. Click Create New.
3. Name: `SSLVPN-to-LAN`.
4. Incoming Interface: SSL-VPN tunnel interface (ssl.root).
5. Outgoing Interface: internal1.
6. Source: `all`, and add the user group `SSLVPN-Users`.
7. Destination: `SSLVPN-Destinations` (not `all`).
8. Schedule: always. Service: ALL. Action: Accept.
9. For this homelab, set NAT to **ON** unless the routed core already has a return route to the `SSLVPN_TUNNEL_ADDR1` client pool through the FortiGate.
10. Click OK.

> NAT ON source-NATs the client to the FortiGate `internal1` address, giving the main LAN and PC VLAN a simple return path. NAT can be OFF in a fully routed design when the core switch and downstream networks already know how to return traffic to the SSL VPN client pool. Leaving NAT off preserves the original VPN-client address in logs.
> If NAT is OFF, add a route on `COREbaba` for the actual `SSLVPN_TUNNEL_ADDR1` subnet through `10.~~.~~.1`; do not enter the address-object name in a Cisco route command. With the homelab NAT setting ON, this extra Cisco route is not needed.
>
> The Destination here (`SSLVPN-Destinations`) does double duty: in Split Tunneling "Based on Policy Destinations" mode, its member subnets are also pushed to the client. Never use `all`, or the client can full-tunnel and lose internet in this homelab.

![Step 6a — Firewall Policy list (Interface Pair View) before adding the SSL-VPN policy](https://github.com/user-attachments/assets/46e6d041-7622-45d2-b75c-9f29e9a0f93a)

![Step 6b — New Policy SSLVPN-TO-LAN: ssl.root to internal1, source all + SSLVPN-Users, destination LAN, NAT on](https://github.com/user-attachments/assets/c7f6a5f3-e8b0-4589-910c-a284e0c1863d)

![Step 6c — The SSLVPN-TO-LAN policy under ssl.root to internal1 in the list](https://github.com/user-attachments/assets/b3a9d02c-d291-4fb4-a9be-f726a54c0104)

### Step 7 — Connect from FortiClient

1. Open FortiClient and add a new SSL-VPN connection.

| Field | Value |
|---|---|
| Type | SSL-VPN |
| Server | 200.0.0.~~ |
| Port | 10443 |
| Username | your user |

2. Save, then Connect.
3. Accept the certificate warning.
4. Enter the password.

<img alt="Step 7 — FortiClient (iOS) SSL-VPN connection: server 200.0.0.~~:10443, port 10443" src="https://github.com/user-attachments/assets/33e0939e-03e4-48b1-a83b-438a08019936" width="340" />

> The client must be able to reach `200.0.0.~~:10443`. Quick check: open `https://200.0.0.~~:10443` in a browser first. If the login page loads, FortiClient will connect.
> Loading the login page proves only that the SSL VPN service is reachable. Use FortiClient for the tunnel configured by this guide.
>
> Force-quit and reconnect FortiClient after any portal change. Routes and DNS load only at connect time.

## Verification

1. With the VPN connected, ping `10.~~.~~.1` (FortiGate LAN). It should reply.
2. From FortiClient, confirm routes exist for `10.~~.~~.0/24` and `10.~~.10.0/24`.
3. Ping the student PC: `ping 10.~~.10.10`.
4. Confirm the client still browses the internet (proves split tunneling is active).

## Troubleshooting

| Symptom (VPN on) | Cause | Fix |
|---|---|---|
| Internet dies completely | Full tunnel (split routes not applied) | Set Split Tunneling to Enabled Based on Policy Destinations, and set the SSLVPN-to-LAN policy destination to `SSLVPN-Destinations`. Trusted Destinations may silently full-tunnel on mobile clients |
| Sites open by IP but not by name, or "Primary DNS is unreachable" | FortiGate pushes an unreachable DNS server | Set DNS Server to "Same as client system DNS" in SSL-VPN Settings, then reconnect |
| Cannot reach LAN, gateway pings | No return path | Add a return route to the SSL VPN client pool, or enable NAT on the ssl.root to internal1 policy for this homelab |
| Cannot ping `10.~~.~~.1` at all | Wrong portal mapped or tunnel mode off | Map user to full-access, enable Tunnel Mode |
| Cannot ping `10.~~.10.10`, and no PC VLAN route appears in FortiClient | PC subnet missing from the pushed routes | Add `PC-VLAN-~~` to `SSLVPN-Destinations`, verify the policy destination, then reconnect FortiClient |
| PC VLAN route is present, but `10.~~.10.10` does not reply | FortiGate route, firewall policy, PC gateway, or Windows Firewall is wrong | Verify the FortiGate route to `10.~~.10.0/24`, policy hit count, PC gateway `10.~~.10.4`, and ICMP allowance |
| FortiGate route to the PC VLAN is missing | Cisco SVI or FortiGate-to-`COREbaba` routing is incomplete | Verify VLAN 10 and routed `Gi0/1`, then complete `OSPF.md` or add the documented static route |
| "Could not enable split tunneling as policy has all" | Enabled Based on Policy Destinations with a policy whose destination is `all` | Set the SSLVPN-to-LAN policy destination to `SSLVPN-Destinations` (not `all`), then enable Based on Policy Destinations |
| Reaches `10.~~.~~.1` but not the internet; SSLVPN-to-LAN policy shows 0 bytes | Client is full-tunneling; reaching the FortiGate LAN IP is local traffic and does not use the policy | Fix split tunneling as above (Policy Destinations + policy destination `SSLVPN-Destinations`) |

## Notes

In a homelab with no real internet behind wan1, the SSL VPN reaches the LAN only. The client's internet comes from its own connection, not the FortiGate. Full tunnel with no upstream will cut the client's internet.

Split Tunneling mode matters on mobile clients. "Based on Policy Destinations" (with the SSLVPN-to-LAN policy destination set to `SSLVPN-Destinations`) pushes the main LAN and PC VLAN routes to the phone. "For Trusted Destinations" did not take effect reliably in the original mobile FortiClient test, so the phone full-tunneled and lost internet. If internet drops the moment the VPN connects, this mode setting is the first thing to check.
