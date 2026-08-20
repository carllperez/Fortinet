<img width="1440" height="867" alt="2" src="https://github.com/user-attachments/assets/bc997ea4-73da-4785-ac51-e74735478076" /># FortiGate 60F — SNMP Monitoring

| | |
|---|---|
| Device | FortiGate 60F and an SNMP manager |
| Firmware | FortiOS 7.0.x |
| Purpose | Poll system/interface data and receive SNMPv2c traps |
| License | No paid FortiGuard subscription required |
| Est. time | 20–30 minutes |

## Overview

SNMP lets a monitoring server query FortiGate counters and receive event traps. FortiGate SNMP access is read-only. This beginner lab uses SNMPv2c because it is widely supported; SNMPv3 is preferred in production because it provides user-based authentication and encryption.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives monitoring server `10.61.61.50`.

## Network overview

| Item | Value |
|---|---|
| FortiGate management IP | `10.~~.~~.1` on `internal1` |
| Monitoring server | `10.~~.~~.50` |
| Query port | UDP `161` |
| Trap destination port | UDP `162` |
| Community | `LAB-RO` example only |

## Prerequisites

- Monitoring server can route directly to `10.~~.~~.1`
- SNMP tools such as LibreNMS, PRTG, Zabbix, or Net-SNMP are ready
- A strong non-default community string is chosen
- The trusted host is a single monitoring IP, not the whole internet

## Configuration

### Step 1 — Enable the SNMP agent

Go to **System > SNMP**. Enable the SNMP Agent and set meaningful Description, Location, and Contact values.

<img width="1440" height="867" alt="1" src="https://github.com/user-attachments/assets/e4b803be-01c5-4930-b1c2-a0261acc4b23" />


### Step 2 — Create the v2c community

Under the SNMP v1/v2c table, click **Create New** and configure:

| Field | Value |
|---|---|
| Community Name | A strong value; `LAB-RO` is only a placeholder |
| Status | Enabled |
| Host IP | `10.~~.~~.50/255.255.255.255` |
| Host Type | Query and Trap, or separate hosts as needed |
| v1 Queries | Disabled |
| v2c Queries | Enabled, port 161 |
| v1 Traps | Disabled |
| v2c Traps | Enabled, destination port 162 |
| Events | Interface, system, HA, and other lab-relevant events |

<img width="1440" height="867" alt="2" src="https://github.com/user-attachments/assets/c0f584b9-1b30-4a3e-a022-3d4e690c5d0b" />

### Step 3 — Permit SNMP on the interface

1. Go to **Network > Interfaces**.
2. Edit `internal1`.
3. Under Administrative Access, enable **SNMP**.
4. Save.

This controls access to the FortiGate itself. A normal transit firewall policy is not what enables SNMP polling of the interface address.

<img width="1440" height="867" alt="3" src="https://github.com/user-attachments/assets/21166b0f-063d-4898-8293-7744f8a9531d" />


## Verification

From a Linux/macOS Net-SNMP host:

```text
snmpwalk -v2c -c <community> 10.~~.~~.1 1.3.6.1.2.1.1
```

The system MIB should return description, uptime, contact, name, and location. Then poll interface counters and confirm they change while traffic passes.

On the FortiGate:

```shell
show system snmp sysinfo
show system snmp community
show system interface internal1
diagnose sniffer packet internal1 'udp port 161 or 162' 4 20 l
```

To test traps, cause a controlled event such as administratively toggling an unused monitored lab interface, then verify receipt on the manager.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Poll times out and no packet reaches FortiGate | Routing or monitoring-host firewall is wrong | Ping first and capture UDP 161 on both ends |
| Request reaches FortiGate but no reply | SNMP not enabled on `internal1`, community differs, or trusted host mask excludes the manager | Enable interface SNMP and match the exact community/host |
| System OIDs work but vendor OIDs do not | Fortinet MIB files are missing from the manager | Import the Fortinet MIB package and dependencies |
| Queries work but traps do not | Host type/events/trap port is wrong or collector is not listening on UDP 162 | Enable v2c traps and selected events; verify listener/firewall |
| Traps show an unexpected source IP | FortiGate chose a different route/source address | Set the intended SNMP host/source design and verify routing |
| Anyone on the LAN can query | Trusted host was configured as a broad subnet | Restrict it to `10.~~.~~.50/32` and migrate to SNMPv3 |

## Notes

- A v2c community is effectively a clear-text shared credential. Never expose UDP 161 on `wan1`.
- SNMPv3 should be used for production monitoring when the manager supports it.
- Interface indexes can change after major interface redesigns; monitor stable identities and re-discover after changes.
