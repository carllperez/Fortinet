# FortiGate 60F — Cisco Collapsed-Backbone Switching

| | |
|---|---|
| Devices | FortiGate 60F, Cisco `COREbaba`, and Cisco `COREtaas` |
| Firmware | FortiOS 7.0.x and Cisco IOS |
| Purpose | Build the Day 1 routed-core VLAN, trunk, EtherChannel, and access-port topology |
| License | No paid FortiGuard subscription required |
| Est. time | 45–60 minutes |

## Overview

This guide follows `DAY1-May5-SirRob.txt`. The Cisco `COREbaba` Layer-3 switch owns the VLAN gateways and routes between VLANs. The FortiGate connects to `COREbaba` through a routed Ethernet link; it does not own VLAN 10 and it is not the switch-to-switch 802.1Q trunk endpoint.

> **Day 1 Cisco foundation lab:** This guide configures `COREtaas-~~` and `COREbaba-~~`. Complete and verify it before starting any guide marked **Day 1 Cisco prerequisite**.

> Replace `~~` with the student's monitor number everywhere. For monitor 61, the student PC is `10.61.10.10/24` and its gateway is `10.61.10.4`.

## Address and VLAN plan

| VLAN | Purpose | `COREtaas` SVI | `COREbaba` gateway |
|---:|---|---|---|
| 1 | Management/data | `10.~~.1.2/24` | `10.~~.1.4/24` |
| 10 | Student PC / wireless | `10.~~.10.2/24` | `10.~~.10.4/24` |
| 50 | IP cameras | `10.~~.50.2/24` | `10.~~.50.4/24` |
| 100 | Voice | `10.~~.100.2/24` | `10.~~.100.4/24` |

| Routed link | Address |
|---|---|
| FortiGate `internal1` | `10.~~.~~.1/24` |
| `COREbaba` `Gi0/1` | `10.~~.~~.4/24` |
| Student PC | `10.~~.10.10/24`, gateway `10.~~.10.4` |

```text
wan1 [ FortiGate ] internal1 10.~~.~~.1
                         |
                  routed Ethernet
                         |
                Gi0/1 10.~~.~~.4
                    [ COREbaba ]
             SVI VLAN 10 10.~~.10.4 ---- student PC 10.~~.10.10
                         |
             Po1: Fa0/10-12, 802.1Q/LACP
                         |
                    [ COREtaas ]
```

> Address ownership is critical: configure `10.~~.10.4` only on `COREbaba` VLAN 10. Do not also create a FortiGate VLAN interface with that address.

## Prerequisites

- `internal1` is removed from any FortiGate hardware/software switch that prevents it from being used as the routed link.
- Both Cisco switches support Layer-3 SVIs, 802.1Q trunks, and LACP EtherChannel.
- Port names are adjusted if the physical switches do not use `Fa0/x` and `Gi0/1`.
- Console access is available before changing management or uplink ports.

## Configuration

### Step 1 — Configure the FortiGate routed link

Under **Network > Interfaces**, configure `internal1` as `10.~~.~~.1/255.255.255.0` and enable Ping for the lab.

Do not create `PC-VLAN10` on the FortiGate for this design. Traffic from all Cisco VLANs arrives untagged on the routed `internal1` link after `COREbaba` performs inter-VLAN routing.

### Step 2 — Configure `COREtaas`

```text
configure terminal
hostname COREtaas-~~
vlan 10
 name WIFIVLAN
vlan 50
 name IPCameraVLAN
vlan 100
 name VOICEVLAN
interface Vlan1
 description MGMTDATA
 ip address 10.~~.1.2 255.255.255.0
 no shutdown
interface Vlan10
 description WIRELESS
 ip address 10.~~.10.2 255.255.255.0
 no shutdown
interface Vlan50
 description IPCCTV
 ip address 10.~~.50.2 255.255.255.0
 no shutdown
interface Vlan100
 description VOICEVLAN
 ip address 10.~~.100.2 255.255.255.0
 no shutdown
end
```

The `.2` addresses identify the upper switch. They are not the client default gateways; client gateways are the `.4` SVIs on `COREbaba`.

### Step 3 — Configure `COREbaba` routing and SVIs

```text
configure terminal
hostname COREbaba-~~
ip routing
vlan 10
 name WIFIVLAN
vlan 50
 name IPCameraVLAN
vlan 69
 name vlanNIrobert
vlan 70
 name EXTRAVLAN
vlan 71
 name HRD-POLICY
vlan 100
 name VOICEVLAN
interface GigabitEthernet0/1
 description ROUTED-TO-FORTIGATE-internal1
 no switchport
 ip address 10.~~.~~.4 255.255.255.0
 no shutdown
interface Vlan1
 description MGMTDATA
 ip address 10.~~.1.4 255.255.255.0
 no shutdown
interface Vlan10
 description STUDENT-PC-WIRELESS
 ip address 10.~~.10.4 255.255.255.0
 no shutdown
interface Vlan50
 description IPCCTV
 ip address 10.~~.50.4 255.255.255.0
 no shutdown
interface Vlan100
 description VOICEVLAN
 ip address 10.~~.100.4 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.~~.~~.1
end
```

The default route sends unknown traffic, including internet traffic, to the FortiGate. `OSPF.md` replaces or supplements the static routing needed when dynamic routing is enabled.

### Step 4 — Build the inter-switch LACP trunk

Run the same configuration on `COREtaas` and `COREbaba`:

```text
configure terminal
interface range FastEthernet0/10-12
 channel-protocol lacp
 channel-group 1 mode active
 no shutdown
interface Port-channel1
 description LACP-TRUNK-COREtaas-COREbaba
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 1,10,50,69,70,71,100
 no shutdown
end
```

Some Cisco models use only 802.1Q and reject `switchport trunk encapsulation dot1q`; omit that single command on those models. All member links must use matching speed, duplex, trunk, and allowed-VLAN settings.

### Step 5 — Place access ports in the required VLANs

On `COREbaba`:

```text
configure terminal
interface FastEthernet0/2
 description STUDENT-PC-OR-WIFI
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
interface FastEthernet0/4
 description STUDENT-PC-OR-WIFI
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
interface FastEthernet0/6
 description IP-CAMERA
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
interface FastEthernet0/8
 description IP-CAMERA
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
interface FastEthernet0/3
 description VOICE-DEVICE
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
interface FastEthernet0/5
 description CISCO-PHONE-AND-PC
 switchport mode access
 switchport access vlan 1
 switchport voice vlan 100
 mls qos trust device cisco-phone
 spanning-tree portfast
interface FastEthernet0/7
 description CISCO-PHONE-AND-PC
 switchport mode access
 switchport access vlan 1
 switchport voice vlan 100
 mls qos trust device cisco-phone
 spanning-tree portfast
end
```

Connect the standard student PC to an access port in VLAN 10 and assign `10.~~.10.10/24`, gateway `10.~~.10.4`.

### Step 6 — Give the FortiGate routes to the VLANs

The preferred Day 1 method is OSPF; follow `OSPF.md`. For a temporary static-routing test, add a FortiGate route:

| Field | Value |
|---|---|
| Destination | `10.~~.0.0/16` |
| Interface | `internal1` |
| Gateway | `10.~~.~~.4` |

The directly connected `10.~~.~~.0/24` route remains more specific than the summary route.

### Step 7 — Create the FortiGate internet policy

Because the Cisco core routes the PC VLAN, the FortiGate policy uses `internal1`, not `PC-VLAN10`:

| Name | Incoming | Outgoing | Source | Destination | NAT |
|---|---|---|---|---|---|
| `Cisco-LANs-to-WAN` | `internal1` | `wan1` | `10.~~.0.0/16` | `all` | On |

Start with only the required source subnets if the lab needs tighter access. Inter-VLAN traffic is routed inside `COREbaba` and does not cross the FortiGate, so FortiGate policies cannot filter it in this topology.

## Verification

On both switches:

```text
show ip interface brief
show vlan brief
show interfaces trunk
show etherchannel summary
show interfaces port-channel 1
```

On `COREbaba`:

```text
show ip route
ping 10.~~.~~.1 source 10.~~.~~.4
ping 10.~~.10.10 source 10.~~.10.4
```

On the FortiGate:

```shell
get router info routing-table details 10.~~.10.10
execute ping-options source 10.~~.~~.1
execute ping 10.~~.~~.4
diagnose sniffer packet internal1 'host 10.~~.10.10' 4 20 l
```

Expected results:

- `Port-channel1` is up and its member ports show bundled state.
- VLAN 10 is active and the PC uses gateway `10.~~.10.4`.
- The FortiGate route to `10.~~.10.0/24` points to `10.~~.~~.4` or is learned through OSPF.
- PC-to-WAN traffic enters the FortiGate on `internal1` and matches the NAT policy.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| PC receives `169.254.x.x` | Access VLAN or DHCP on `COREbaba` is wrong | Check the access port, VLAN 10 SVI, and `DHCP.md` |
| SVI is down/down | No active Layer-2 port exists in that VLAN | Enable an access/trunk member carrying the VLAN |
| Port-channel is suspended or amber | Member settings, LACP mode, or cabling do not match | Compare both ends and verify `show etherchannel summary` |
| PC reaches `.4` but not the FortiGate | Routed `Gi0/1`, `ip routing`, or default route is missing | Verify `Gi0/1`, `show ip route`, and ping `10.~~.~~.1` |
| FortiGate reaches `COREbaba` but not the PC | FortiGate lacks a VLAN route or the PC firewall blocks traffic | Add OSPF/static route, verify PC gateway, and allow the test traffic |
| Internet policy has zero hits | Policy incorrectly expects `PC-VLAN10` | Use incoming interface `internal1` for the routed-core design |
| Duplicate-IP or unstable ARP | `.4` is configured on both FortiGate and switch | Keep every VLAN `.4` gateway only on `COREbaba` |

## Notes

- Use `write memory` or `copy running-config startup-config` after verification.
- The Day 1 source names VLAN 10 `WIRELESS`; this repository also places the standard student PC in that VLAN.
- Use `FortiLink.md` only for FortiSwitch management. FortiLink is a different design from this Cisco collapsed backbone.
