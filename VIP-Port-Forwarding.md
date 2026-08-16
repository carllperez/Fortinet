# FortiGate 60F — VIP Port Forwarding

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Publish an internal TCP/80 web server as WAN TCP/8080 |
| License | No paid FortiGuard subscription required |
| Est. time | 20–30 minutes |

## Overview

A Virtual IP (VIP) matches a destination on the WAN and performs destination NAT. This lab maps `200.0.0.~~:8080` to `10.~~.~~.10:80`. The VIP does not permit traffic by itself; a WAN-to-LAN firewall policy must reference it as the destination.

> Replace `~~` with the student's monitor number. Example: monitor 61 gives WAN `200.0.0.61` and server `10.61.61.10`.

```text
External PC ──> 200.0.0.~~:8080 [ FortiGate ] ──> 10.~~.~~.10:80
                       DNAT external TCP 8080 to mapped TCP 80
```

## Prerequisites

- Web server `10.~~.~~.10/24`, gateway `10.~~.~~.1`, listening on TCP 80
- FortiGate `internal1` is `10.~~.~~.1/24`
- `wan1` owns or receives traffic for `200.0.0.~~`
- External test device is not on the internal LAN
- Upstream router forwards the public address/port if the FortiGate is behind another NAT

## Configuration

### Step 1 — Prove the server works internally

From another LAN device, open `http://10.~~.~~.10`. If this fails, fix the server service, local firewall, IP, and gateway before creating the VIP.

### Step 2 — Create the VIP

1. Go to **Policy & Objects > Virtual IPs**.
2. Click **Create New > Virtual IP**.
3. Configure:

| Field | Value |
|---|---|
| Name | `VIP-Web-8080` |
| Interface | `wan1` |
| External IP Address/Range | `200.0.0.~~` |
| Mapped IP Address/Range | `10.~~.~~.10` |
| Port Forwarding | Enabled |
| Protocol | TCP |
| External Service Port | `8080` |
| Map to Port | `80` |

> **Screenshot:** Virtual IP showing external TCP 8080 mapped to server TCP 80.

Using `wan1` limits the VIP to the intended ingress interface. If the external IP is not assigned to the FortiGate itself, the upstream network must still route or ARP that address toward it.

### Step 3 — Create the WAN-to-LAN policy

Under **Policy & Objects > Firewall Policy**, create:

| Field | Value |
|---|---|
| Name | `WAN-to-Web-8080` |
| Incoming Interface | `wan1` |
| Outgoing Interface | `internal1` |
| Source | A specific external test subnet if possible |
| Destination | `VIP-Web-8080` |
| Schedule | `always` |
| Service | `HTTP` (TCP 80, the mapped service) |
| Action | Accept |
| NAT | Off |
| Log Allowed Traffic | All Sessions |

The VIP performs DNAT before the forward-policy service check, so the narrow policy service is the mapped/internal service, TCP 80. Do not use a custom TCP/8080 policy service for this 8080-to-80 mapping; that policy will not match after DNAT. Policy NAT is normally off so the server sees the real external source. The server must return through the FortiGate; otherwise enable source NAT only as a deliberate workaround for an asymmetric return path.

> Gotcha: the policy destination is the VIP object, not the server's private address object.

## Testing

From an external network:

```text
http://200.0.0.~~:8080
```

Do not rely on testing the public address from the same LAN. Hairpin/loopback access is a separate design with additional policy and DNS considerations.

## Verification

Check **Log & Report > Forward Traffic** for policy `WAN-to-Web-8080`. Then capture both sides:

```shell
diagnose sniffer packet any 'host 10.~~.~~.10 or port 8080' 4 30 l
```

You should see the inbound connection to WAN TCP 8080 and translated traffic to `10.~~.~~.10:80`, followed by replies.

Use flow debug for one external source if needed:

```shell
diagnose debug reset
diagnose debug flow filter dport 8080
diagnose debug flow trace start 20
diagnose debug enable
```

Stop with `diagnose debug disable` and `diagnose debug reset`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No packet reaches `wan1` | Upstream routing/NAT, ISP filtering, or carrier-grade NAT | Test the upstream path and forward TCP 8080 to the FortiGate where possible |
| Packet reaches WAN but policy counter stays zero | Wrong VIP external IP/interface/port or policy destination | Match the VIP exactly and reference it in the WAN-to-LAN policy |
| VIP matches but the forward policy still does not | Policy service was set to external TCP 8080 instead of mapped TCP 80 | Select built-in `HTTP` for the 8080-to-80 VIP |
| SYN reaches server but no reply returns | Server gateway or host firewall is wrong | Set gateway to `10.~~.~~.1` and allow TCP 80 from the test source |
| Server replies through another router | Asymmetric routing | Correct the server/route design; use policy SNAT only if preserving the source is not required |
| Internal test works but external test times out | Missing upstream exposure or WAN policy | Capture on `wan1`, then check VIP and policy in that order |
| External port 80 works but 8080 does not | External/mapped port values were reversed | External is 8080; mapped server port is 80 |

## Security considerations

- Restrict Source to known test addresses whenever possible.
- Publish only the required port, patch the web server, and review logs.
- A VIP exposes a service; it does not make an unsafe service safe.
- Remove or disable the policy after the demonstration if the service is no longer needed.
