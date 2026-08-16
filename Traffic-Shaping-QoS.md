# FortiGate 60F — Traffic Shaping and QoS

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Limit camera bandwidth and prioritize important lab traffic |
| License | No paid FortiGuard subscription required |
| Est. time | 30–45 minutes |

## Overview

Traffic shaping controls bandwidth after a firewall policy has accepted a session. A shared shaper divides one bandwidth allowance among all matching sessions. A per-IP shaper gives each matching source IP its own allowance. Priority and guaranteed bandwidth matter when traffic competes for a constrained egress link.

> **Day 1 Cisco prerequisite:** This lab uses the Day 1 camera and student-PC VLANs. Configure and verify `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) before applying shaping policies.

> Complete `VLAN.md` and `Firewall-Policies-and-NAT.md` first.
>
> Replace `~~` with the student's monitor number. Example: monitor 61 gives student PC `10.61.10.10`.

## Lab goals

| Traffic | Source | Result |
|---|---|---|
| Camera internet | `10.~~.50.0/24` | Shared maximum of 5 Mbps |
| Camera per-device test | Same VLAN | Optional 1 Mbps per source IP |
| Important HTTPS | `10.~~.10.0/24` (student PC `10.~~.10.10`) | Higher priority/guaranteed share during congestion |

## Prerequisites

- Working `internal1`-to-WAN policies for the Cisco-routed VLANs
- FortiGate route to the Cisco VLANs through `COREbaba` at `10.~~.~~.4`
- A measured or deliberately simulated WAN upload rate
- Traffic Shaping enabled under **System > Feature Visibility**
- Test tools capable of creating sustained traffic

## Configuration

### Step 1 — Create the shared camera shaper

1. Go to **Policy & Objects > Traffic Shaping**.
2. Open the **Shared Shapers** tab and click **Create New**.
3. Name it `Cameras-5Mbps`.
4. Set Maximum Bandwidth to `5000 Kbps`.
5. Leave Guaranteed Bandwidth at `0` for a strict camera cap.
6. Set Priority to Low and save.

> **Screenshot:** Shared shaper `Cameras-5Mbps` with a 5000 Kbps maximum.

### Step 2 — Apply it with a shaping policy

In **Policy & Objects > Traffic Shaping**, open **Traffic Shaping Policies** and create:

| Field | Value |
|---|---|
| Source | `IPCCTV-NET` (`10.~~.50.0/24`) |
| Destination | `all` |
| Service | `ALL` |
| Outgoing Interface | `wan1` |
| Shared Shaper | `Cameras-5Mbps` |

Place specific shaping rules above general ones. FortiGate first accepts traffic with a firewall policy and then looks for the first matching shaping policy.

### Step 3 — Compare a per-IP shaper

Create a Per-IP Shaper named `Camera-1Mbps-Each` with Maximum Bandwidth `1000 Kbps`. Temporarily replace the shared shaper action with this per-IP shaper.

- Two cameras under the shared 5 Mbps shaper compete for one 5 Mbps total.
- Two cameras under the 1 Mbps per-IP shaper can each approach 1 Mbps.

Do not apply both just to make the lab look more advanced; test one behavior at a time.

### Step 4 — Prioritize important traffic

For a simple FortiOS 7.0 lab, create another shared shaper such as `Important-High` with a nonzero guaranteed bandwidth, a maximum appropriate to the link, and High priority. Apply it in a more-specific shaping policy matching the important source and HTTPS service.

Guaranteed bandwidth and priority cannot create bandwidth the WAN does not have. They influence allocation only when the egress queue is congested. The sum of guarantees must remain realistic.

For class-based interface shaping, create traffic classes and a Traffic Shaping Profile, assign class IDs in shaping policies, then edit `wan1` under **Network > Interfaces** to set the real Outbound Bandwidth and apply the profile. This is more accurate for several competing classes but is optional for this beginner lab.

## Verification

1. Start a large transfer from one camera-VLAN test client and record throughput.
2. Start a second transfer and compare shared versus per-IP behavior.
3. Saturate the test WAN, then generate the important traffic and observe its latency/throughput.
4. Check policy and shaper statistics in **FortiView > Traffic Shaping** where available.

```shell
show firewall shaper traffic-shaper
show firewall shaper per-ip-shaper
show firewall shaping-policy
diagnose firewall shaper traffic-shaper list
```

The shaping policy must show the expected interface and shaper. Measured throughput will include protocol overhead and may not equal the configured number exactly.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Throughput is not limited | Shaping policy does not match source/service/outgoing interface or sits below a broad rule | Check counters and rule order |
| Each device gets 5 Mbps instead of sharing it | Per-IP rather than shared shaper is applied | Use `Cameras-5Mbps` as the shared shaper action |
| All devices together get only 1 Mbps | Shared shaper used where per-IP behavior was intended | Apply `Camera-1Mbps-Each` as a per-IP shaper |
| High priority makes no visible difference | Link is not congested or outbound bandwidth is not modeled correctly | Saturate the egress link and set accurate interface bandwidth for class-based shaping |
| Download test is unaffected | Only the forward/upload direction is shaped | Configure a reverse shaper deliberately and understand the traffic direction |
| Existing test behaves differently after edits | Existing session retained old shaping state | Start a new test session after the policy change |

## Notes

- Traffic shaping itself does not require FortiGuard. Application-category or URL-category matching can depend on inspection data and is intentionally omitted.
- Shape slightly below the actual bottleneck so FortiGate, not the upstream modem, controls the queue.
- Start with limits and measurements before adding several classes and priorities.
