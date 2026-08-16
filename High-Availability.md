# FortiGate 60F — Active-Passive High Availability

| | |
|---|---|
| Devices | Two identical FortiGate 60F units |
| Firmware | Same FortiOS 7.0.x build on both |
| Purpose | Build an FGCP active-passive cluster and perform a controlled failover |
| License | No paid FortiGuard subscription required |
| Est. time | 45–60 minutes |

## Overview

FortiGate Clustering Protocol (FGCP) makes two FortiGates operate as one logical firewall. The primary forwards traffic and synchronizes configuration and, when enabled, eligible sessions to the secondary. The secondary takes over the cluster MAC/IP identities after a failure.

## Prerequisites

- Two FortiGate 60F units with the same hardware revision where possible
- Identical FortiOS build and compatible licensing state
- Direct heartbeat cables between two dedicated physical interfaces
- Matching LAN/WAN cabling from both units to the same Layer-2 networks
- Console access to both units and backups before clustering

> The 60F has limited ports. Use interfaces labeled for HA where present and a second unused physical interface removed from any hardware/software switch. Heartbeat cannot use a VLAN subinterface, IPsec interface, aggregate, or a port that remains a switch member.

## Topology

```text
                    heartbeat-1
             FG-60F-A =========== FG-60F-B
                    heartbeat-2
               |                     |
          same WAN network       same WAN network
               |                     |
          same LAN network       same LAN network
```

## Configuration

### Step 1 — Prepare each unit separately

1. Upgrade/downgrade both units to the exact same approved FortiOS 7.0.x build.
2. Give them unique hostnames such as `FGT60F-A` and `FGT60F-B`.
3. Remove the chosen heartbeat ports from other interface groups and references.
4. Configure the intended primary fully; keep the secondary isolated from production data networks until HA settings are ready.

### Step 2 — Configure FGCP on the intended primary

Go to **System > HA** and configure:

| Field | Value |
|---|---|
| Mode | Active-Passive |
| Group Name | `LAB-HA` |
| Password | Same strong HA password on both units |
| Device Priority | `200` on unit A |
| Heartbeat Interfaces | Two dedicated physical ports, priority `100` then `50` |
| Session Pickup | Enabled |
| Monitor Interfaces | Critical LAN and WAN data interfaces |
| Override | Disabled for the first lab |

Group ID is configured in CLI and must match:

```shell
config system ha
    set group-id 10
end
```

> **Screenshot:** System > HA on unit A showing Active-Passive mode and both heartbeat links.

### Step 3 — Configure the secondary

Configure the same Mode, Group Name, Group ID, Password, and heartbeat interfaces on unit B. Set Device Priority to `100`. Connect the heartbeat cables directly, then wait for discovery and synchronization before connecting both units to the data networks.

Do not copy a complete production configuration onto the secondary after it joins. The cluster primary synchronizes most configuration automatically.

### Step 4 — Understand election settings

With override disabled, the cluster favors stability; a recovered unit with higher configured priority does not automatically preempt a healthy primary merely because it returned. With override enabled, configured priority has stronger election influence and can cause planned preemption. Keep override disabled until election behavior has been tested.

Primary election also considers monitored-interface state and other health/uptime factors. Device priority is not the only input.

## Verification

```shell
get system ha status
diagnose sys ha checksum cluster
diagnose sys ha history read
```

Confirm:

- Both serial numbers appear.
- One member is primary and one secondary.
- Both heartbeat interfaces are up.
- Configuration checksums match after synchronization settles.

Use `execute ha manage <member-index> <admin-name>` from the primary to reach the secondary CLI without assigning normal data-interface IPs to it.

## Controlled failover test

1. Start a continuous ping through the cluster and an ordinary TCP session.
2. Record `get system ha status` and the current primary serial number.
3. Disconnect a monitored data interface on the primary, or power off the primary during a maintenance window.
4. Observe a short traffic interruption while the secondary assumes the cluster identity and upstream switches relearn MAC locations.
5. Confirm the former secondary is now primary and traffic resumes.
6. Restore the failed unit and watch it rejoin as secondary and resynchronize.

Session pickup can preserve many TCP/UDP sessions, but not every application/session type survives. A few lost pings and reconnects are realistic. Pulling both heartbeat links while both units remain attached to data networks risks split brain and must not be used as a casual failover test.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Units never form one cluster | Firmware build, group name, group ID, password, or heartbeat ports differ | Compare all HA settings locally on both consoles |
| Heartbeat port cannot be selected | It is still a switch/aggregate member or referenced elsewhere | Remove the physical port from those objects first |
| Checksums remain different | Synchronization is incomplete or an unsynchronized/per-device setting differs | Wait, inspect checksum output, and verify firmware/VDOM alignment |
| Both units become primary | Heartbeat connectivity is lost (split brain) | Restore heartbeat isolation immediately and verify direct cables |
| Failover does not occur when WAN cable is removed | WAN is not in Monitor Interfaces | Add the critical data interface to monitoring on the cluster |
| Recovered unit unexpectedly takes primary role | Override/election settings differ from the intended behavior | Disable override for stable non-preemptive behavior or document deliberate preemption |
| GUI becomes briefly unreachable during cluster formation | FGCP changes interface MAC ownership and negotiates roles | Wait for convergence and reconnect to the cluster IP |

## Notes

- HA protects against a unit/interface failure; it does not replace dual WAN, switch redundancy, backups, or configuration review.
- Configuration synchronization is not a backup. Export cluster backups separately.
- Rejoining can generate substantial heartbeat synchronization traffic; use dedicated direct links.

