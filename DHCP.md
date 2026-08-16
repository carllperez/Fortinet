# FortiGate 60F — DHCP Server, Reservations, and Relay

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Assign correct client settings on LANs and relay requests to an external server |
| License | No paid FortiGuard subscription required |
| Est. time | 25–40 minutes |

## Overview

FortiGate can run a separate DHCP server on each routed interface or relay broadcasts to an external DHCP server. A scope must match the interface subnet and provide a usable gateway, DNS server, lease time, and non-overlapping address range.

> Complete `VLAN.md` first if you want one scope per VLAN.
>
> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

## Network overview

| Scope | Interface | Gateway | Dynamic range |
|---|---|---|---|
| LAN | `internal1` | `10.~~.~~.1` | `10.~~.~~.100–200` |
| PCs | `PC-VLAN10` | `10.~~.10.4` | `10.~~.10.100–199` |
| Guest | `GUEST-VLAN20` | `10.~~.20.1` | `10.~~.20.100–199` |

## Configuration

### Step 1 — Configure a DHCP server

1. Go to **Network > Interfaces**.
2. Edit `internal1` and enable DHCP Server.
3. Set the range to `10.~~.~~.100–10.~~.~~.200`.
4. Set mask `255.255.255.0`.
5. Set Default Gateway to Same as Interface IP.
6. Use Same as System DNS or specify reachable DNS servers.
7. Set a lab lease time such as one day.
8. Save.

> **Screenshot:** internal1 DHCP server showing range, gateway, DNS, and lease time.

Keep infrastructure addresses outside the dynamic pool. Do not create overlapping ranges on the same broadcast domain.

### Step 2 — Reserve an address by MAC

Go to **Dashboard > Network**, expand the DHCP widget, locate a current lease, and choose **Create DHCP Reservation**. Alternatively, edit the DHCP server's reserved-address list and enter:

| Field | Example |
|---|---|
| Description | `Lab-Printer` |
| MAC address | Device's actual MAC |
| Reserved IP | `10.~~.10.10` for the standard student PC |

Renew the client lease. The student PC should receive `10.~~.10.10/24` with gateway `10.~~.10.4`. The reservation must be in the interface subnet and must not collide with another static device. Keeping reservations outside the dynamic range makes conflicts easier to prevent.

### Step 3 — Add scopes to VLANs

Edit `PC-VLAN10` and `GUEST-VLAN20` under **Network > Interfaces** and enable their separate DHCP servers using the table above. DHCP broadcasts do not cross VLANs, so each VLAN needs its own server or relay.

### Step 4 — Configure DHCP relay instead of a local server

Use relay on an interface only when an external server, such as `10.~~.~~.20`, owns the scope.

1. Edit the client-facing interface under **Network > Interfaces**.
2. Expand Advanced DHCP settings and set Mode to **Relay**.
3. Enter DHCP Server IP `10.~~.~~.20`.
4. Save.

CLI equivalent for a VLAN example:

```shell
config system interface
    edit "GUEST-VLAN20"
        set dhcp-relay-service enable
        set dhcp-relay-ip "10.~~.~~.20"
    next
end
```

Disable/remove the local DHCP server on that interface before using the relay-only lab. The external server needs a scope for `10.~~.20.0/24`, gateway `10.~~.20.1`, and a route back to that subnet through the FortiGate. The relay inserts gateway information so the server can select the correct scope.

## Verification

On a client, release and renew the lease. Verify address, mask, gateway, DNS, and lease duration—not just the IP address.

On FortiGate:

```shell
execute dhcp lease-list
show system dhcp server
diagnose sniffer packet any 'udp port 67 or 68' 4 30 l
```

A normal exchange shows Discover, Offer, Request, and Acknowledgment. For relay, the capture should also show unicast relay traffic between the FortiGate and external server.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Client gets `169.254.x.x` (APIPA) | No DHCP offer reaches it | Check link, access VLAN, trunk tags, scope status, and capture UDP 67/68 |
| Client gets an address but wrong gateway | Scope option points to another router | Set gateway to the client-facing FortiGate interface |
| Client receives an address from another subnet | Wrong access VLAN, native VLAN leak, or rogue DHCP server | Inspect switch port VLAN and capture the offer's source |
| Some clients work and new clients fail | DHCP range is exhausted | Review leases, enlarge the valid pool, or shorten an excessive lease time |
| Reserved client gets a different address | MAC differs because of Wi-Fi private/randomized MAC or stale lease | Reserve the MAC actually presented and renew the lease |
| Relay sends requests but no offer returns | External scope, server route, server firewall, or relay authorization is missing | Create the matching scope and route `10.~~.20.0/24` through the FortiGate |
| Offer returns to FortiGate but not client | VLAN/interface selection is wrong | Verify the client-facing relay interface and switch tagging |

## Notes

- DHCP server and DHCP relay are infrastructure features and do not require FortiGuard licensing.
- A FortiGate firewall policy is not a substitute for correct relay routing and external-server scope configuration.
- For critical devices, document reservations and exclude their addresses from general allocation.
