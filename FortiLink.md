# FortiGate 60F — FortiLink with a Managed FortiSwitch

| | |
|---|---|
| Devices | FortiGate 60F and a compatible FortiSwitch |
| Firmware | FortiOS 7.0.x with compatible FortiSwitchOS |
| Purpose | Discover, authorize, and manage switch ports/VLANs from FortiGate |
| License | No paid FortiGuard security subscription required for local management |
| Est. time | 35–50 minutes |

## Overview

FortiLink is a Fortinet management and data-plane relationship. FortiGate discovers and controls the FortiSwitch, provisions managed VLANs, and assigns them to switch ports. It carries 802.1Q data, but it is not the same as manually configuring a normal trunk between two independent devices.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

## Topology

```text
wan1 [ FortiGate 60F ] FortiLink A/B ===== FortiSwitch FortiLink-capable port
                                              | access port: PC VLAN
                                              | access port: Guest VLAN
                                              | trunk port: native + allowed VLANs
```

## Prerequisites

- FortiSwitch model/firmware is compatible with the FortiOS 7.0.x patch
- Switch Controller is enabled under **System > Feature Visibility**
- FortiGate and FortiSwitch configuration backups
- A console/recovery plan if existing port roles change
- The selected 60F ports are not required for another LAN, HA, or WAN design

## Configuration

### Step 1 — Prepare the FortiLink interface

FortiGate 60F units commonly expose ports `A` and `B` for FortiLink and may already have a `fortilink` interface. Check **Network > Interfaces** and **WiFi & Switch Controller > FortiLink Interface** before creating anything.

1. Edit the existing FortiLink interface.
2. Add the intended physical member or members.
3. Keep Addressing Mode **Dedicated to FortiSwitch**.
4. Keep the FortiLink DHCP service enabled so the switch receives a management address.
5. Enable Automatically Authorize Devices only in a physically controlled lab; otherwise leave it off for manual authorization.
6. Use FortiLink split-interface only when the chosen topology requires it.

If a desired port is unavailable, remove it from the existing hardware-switch Interface Members first. Do not delete the working management interface blindly.

> **Screenshot:** FortiLink Interface showing member ports and Dedicated to FortiSwitch mode.

### Step 2 — Connect and authorize the FortiSwitch

1. Power on the compatible FortiSwitch in a known/default or FortiLink-ready state.
2. Connect the selected FortiGate FortiLink port to a FortiSwitch port that supports FortiLink discovery.
3. Wait several minutes.
4. Go to **WiFi & Switch Controller > Managed FortiSwitch**.
5. Select the discovered switch and click **Authorize** if automatic authorization is disabled.
6. Wait until status is Online/Authorized before creating user VLANs.

> **Screenshot:** Managed FortiSwitch list showing the switch online and authorized.

### Step 3 — Create managed VLANs

Go to **WiFi & Switch Controller > FortiSwitch VLANs** and create:

| VLAN | Name | Gateway | DHCP pool |
|---:|---|---|---|
| 10 | `FSW-PC10` | `10.~~.10.4/24` | `10.~~.10.100–199` |
| 20 | `FSW-GUEST20` | `10.~~.20.1/24` | `10.~~.20.100–199` |

Set the FortiLink interface as the VLAN parent. Enable DHCP on each managed VLAN and create the same inter-VLAN/WAN policies described in `VLAN.md`.

Reserve `10.~~.10.10` for the student PC's MAC address, or configure that address statically with mask `/24` and gateway `10.~~.10.4`.

### Step 4 — Assign access and trunk ports

Go to **WiFi & Switch Controller > FortiSwitch Ports**.

- For a PC access port, set Native VLAN to `FSW-PC10` and do not add unnecessary allowed VLANs.
- For a guest access port, set Native VLAN to `FSW-GUEST20`.
- For a downstream tagged device, choose the intended Native VLAN for untagged frames and add only required tagged networks under Allowed VLANs.

Native VLAN handles untagged ingress and normally leaves untagged on egress. Allowed VLANs are the tagged VLANs accepted/transmitted on that port.

> **Screenshot:** FortiSwitch Ports showing access-port native VLANs and one deliberate trunk allowed-VLAN list.

## Verification

In the GUI, verify topology, switch Online status, port link state, native VLAN, allowed VLANs, and learned client/MAC information.

Useful CLI:

```shell
show system interface fortilink
show switch-controller managed-switch
diagnose switch-controller switch-info port-stats
diagnose switch-controller switch-info lldp
```

Connect a client to each access port and confirm it receives the DHCP scope for that native VLAN. Test the gateway, permitted internet access, and the intended inter-VLAN restriction.

## FortiLink versus a normal trunk

| FortiLink | Normal 802.1Q trunk |
|---|---|
| FortiGate manages the FortiSwitch | Each device is configured independently |
| Discovery/authorization relationship | No Fortinet management relationship |
| Managed VLANs are provisioned through controller | VLANs and trunks are manually matched on both devices |
| Switch status/ports visible in FortiGate | Switch has its own management plane |

Do not configure the same FortiSwitch as both a standalone manually managed trunk and a FortiLink-managed switch at the same time.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Switch never appears | Wrong cable/port, incompatible firmware, switch not FortiLink-ready, or controller hidden | Check compatibility, link, LLDP, feature visibility, and switch state |
| Switch appears but remains unauthorized | Automatic authorization is off | Review the serial number and authorize it manually |
| FortiLink member port is unavailable | It belongs to a hardware switch or another object | Remove references and use a free physical port |
| Authorized switch repeatedly disconnects | Firmware mismatch, bad link, topology loop, or FortiLink settings disagree | Check compatibility matrix, port errors, and topology |
| Client gets no DHCP | Wrong native VLAN, VLAN DHCP disabled, or port link is down | Inspect port assignment and managed VLAN interface/scope |
| Tagged downstream VLAN fails | VLAN absent from Allowed VLANs | Add only the required tagged VLAN and verify the downstream tag |
| Management is lost after port changes | The active management port was repurposed | Use console and restore the original member/interface role |

## Notes

- Firmware compatibility is separate from FortiGuard licensing and must be checked before upgrades.
- FortiLink management does not remove the need for firewall policies between VLANs or to the internet.
- Back up both the FortiGate configuration and any standalone FortiSwitch configuration before adoption.
