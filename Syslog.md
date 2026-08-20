# FortiGate 60F — External Syslog

| | |
|---|---|
| Device | FortiGate 60F and Linux/Windows syslog collector |
| Firmware | FortiOS 7.0.x |
| Purpose | Send traffic and event logs to an external server |
| License | No paid FortiGuard subscription required |
| Est. time | 20–30 minutes |

## Overview

Local memory logs are useful for immediate troubleshooting but are finite and disappear during some resets or long retention periods. A remote syslog collector stores FortiGate traffic and system events outside the appliance.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives syslog server `10.61.61.60`.

## Network overview

| Item | Value |
|---|---|
| Syslog server | `10.~~.~~.60` |
| Transport | UDP 514 for the beginner lab |
| Source interface/IP | `internal1` / `10.~~.~~.1` |
| Minimum severity | Information |
| Format | Default FortiGate syslog |

UDP is easy to demonstrate but does not confirm delivery. Use reliable/TLS logging where the production collector and FortiOS design support it.

## Prerequisites

- rsyslog/syslog-ng on Linux or a Windows syslog collector is listening on UDP 514
- Host firewall permits the FortiGate source IP
- FortiGate can route to the collector
- Firewall policies that should generate traffic logs have logging enabled

## Configuration

### Step 1 — Prepare the collector

Configure the collector to listen on UDP 514 and store messages from `10.~~.~~.1` in a dedicated file or source. Confirm with a local test message before blaming the FortiGate.

<img width="1440" height="867" alt="1" src="https://github.com/user-attachments/assets/e85ba67f-e2ac-48e8-8867-e8156edfa9c1" />
<img width="1440" height="867" alt="2" src="https://github.com/user-attachments/assets/6744fd02-242d-4bc7-ae8c-80669bd49964" />



### Step 2 — Configure remote syslog

Where exposed, go to **Log & Report > Log Settings**, enable Send Logs to Syslog, and enter the server, port, facility, and severity.

The following FortiOS 7.0 CLI makes the source path explicit:

```shell
config log syslogd setting
    set status enable
    set server "10.~~.~~.60"
    set mode udp
    set port 514
    set facility local7
    set source-ip "10.~~.~~.1"
    set format default
    set interface-select-method specify
    set interface "internal1"
end
config log syslogd filter
    set severity information
    set forward-traffic enable
    set local-traffic enable
end
```

If the selected FortiOS 7.0 patch does not expose interface selection in the GUI, use the CLI and confirm with `show log syslogd setting`.

### Step 3 — Enable policy logging

Under **Policy & Objects > Firewall Policy**, edit the lab internet policy and set **Log Allowed Traffic** to **All Sessions**. A syslog filter cannot send a forward-traffic record that the policy never generated.

<img width="1440" height="867" alt="3" src="https://github.com/user-attachments/assets/1d0ada53-3049-4c55-ba7e-22d44311894c" />


## Verification

1. From a client, open a new website through the logged firewall policy.
2. Make a harmless configuration change such as updating a test object's comment.
3. Confirm one forward-traffic log and one system/configuration event reach the collector.

On FortiGate:

```shell
show log syslogd setting
show log syslogd filter
diagnose sniffer packet internal1 'host 10.~~.~~.60 and udp port 514' 4 20 l
```

Traffic logs should include source/destination, interfaces, policy ID, action, and byte counts. Event logs should identify the administrator and changed object where applicable.

## Local versus remote logs

| Local logs | Remote syslog |
|---|---|
| Fast GUI inspection | Longer retention and centralized search |
| Limited by appliance storage/memory | Depends on collector storage |
| Available even if collector path is down | Preserved outside FortiGate |
| Easy for one device | Correlates several network devices |

Use both when practical; remote logging is not proof that every UDP message was delivered.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No syslog packets leave | Remote logging is disabled or no eligible log is generated | Check settings/filter and enable policy logging |
| Packets leave but collector shows nothing | Collector listener, host firewall, facility parsing, or source allow-list is wrong | Capture on the server and inspect collector configuration |
| Event logs arrive but traffic logs do not | Firewall policy logging is off | Set Log Allowed Traffic to All Sessions and create a new session |
| Logs use an unexpected source IP/path | Automatic route selection chose another interface | Set source IP and interface explicitly |
| Collector shows messages but timestamps differ | NTP/timezone differs | Synchronize FortiGate and collector time |
| UDP packet loss during bursts | UDP has no delivery acknowledgement | Use a reliable/TLS mode supported by the collector |

## Notes

- Do not send credentials or private logs across an untrusted network using plain UDP syslog.
- `syslogd2`, `syslogd3`, and `syslogd4` are additional destinations; this lab uses the first `syslogd` instance only.
- External syslog does not require FortiAnalyzer or a FortiGuard subscription.
