# DHCP for the Cisco Core and FortiGate Alternatives

| | |
|---|---|
| Devices | Cisco `COREbaba` and FortiGate 60F |
| Purpose | Assign correct client settings without creating competing DHCP servers |
| License | No paid subscription required |
| Est. time | 25–40 minutes |

## Overview

Choose the DHCP design that matches the gateway owner:

- In the Day 1 Cisco collapsed-backbone design from `VLAN.md`, `COREbaba` owns the `.4` SVIs and should provide DHCP locally, as shown in the supplied switching notes.
- In a standalone FortiGate VLAN lab, the FortiGate may provide DHCP only when it owns the client-facing VLAN interface and gateway.

> **Day 1 Cisco prerequisite:** For the supplied `DAY1-May5-SirRob.txt` topology, configure and verify both `COREtaas-~~` and `COREbaba-~~` with [VLAN.md](VLAN.md) before starting this lab.

Never enable both DHCP servers on the same VLAN. Do not configure FortiGate DHCP on `10.~~.10.0/24` while `COREbaba` owns `10.~~.10.4` unless a deliberately tested relay design is in place.

> Replace `~~` with the student's monitor number. The standard PC remains `10.~~.10.10/24`, gateway `10.~~.10.4`.

## Day 1 Cisco scope plan

| VLAN | Network | Gateway | DNS | Dynamic addresses after exclusions |
|---:|---|---|---|---|
| 1 | `10.~~.1.0/24` | `10.~~.1.4` | `10.~~.1.10` | `10.~~.1.101–199` |
| 10 | `10.~~.10.0/24` | `10.~~.10.4` | `10.~~.1.10` | `10.~~.10.101–199` |
| 50 | `10.~~.50.0/24` | `10.~~.50.4` | `10.~~.1.10` | `10.~~.50.101–199` |
| 100 | `10.~~.100.0/24` | `10.~~.100.4` | `10.~~.1.10` | `10.~~.100.101–199` |

The source notes exclude `.1–.100`. This guide also excludes `.200–.254` to keep the documented dynamic range bounded at `.101–.199`. The PC address `.10`, switch addresses, server addresses, phones, and cameras remain outside the dynamic pool.

## Configuration

### Step 1 — Configure DHCP on `COREbaba`

```text
configure terminal
ip dhcp excluded-address 10.~~.1.1 10.~~.1.100
ip dhcp excluded-address 10.~~.1.200 10.~~.1.254
ip dhcp excluded-address 10.~~.10.1 10.~~.10.100
ip dhcp excluded-address 10.~~.10.200 10.~~.10.254
ip dhcp excluded-address 10.~~.50.1 10.~~.50.100
ip dhcp excluded-address 10.~~.50.200 10.~~.50.254
ip dhcp excluded-address 10.~~.100.1 10.~~.100.100
ip dhcp excluded-address 10.~~.100.200 10.~~.100.254
ip dhcp pool MGMTDATA
 network 10.~~.1.0 255.255.255.0
 default-router 10.~~.1.4
 domain-name MGMTDATA.COM
 dns-server 10.~~.1.10
ip dhcp pool WIFIDATA
 network 10.~~.10.0 255.255.255.0
 default-router 10.~~.10.4
 domain-name WIFIDATA.COM
 dns-server 10.~~.1.10
ip dhcp pool IPCCTV
 network 10.~~.50.0 255.255.255.0
 default-router 10.~~.50.4
 domain-name IPCCTV.COM
 dns-server 10.~~.1.10
ip dhcp pool VOICEVLAN
 network 10.~~.100.0 255.255.255.0
 default-router 10.~~.100.4
 domain-name VOICEVLAN.COM
 dns-server 10.~~.1.10
 option 150 ip 10.~~.100.8
end
```

Option 150 tells supported Cisco phones where to find the call manager/TFTP service.

### Step 2 — Keep the student PC at the standard address

Use one of these approaches:

1. Configure the PC statically as `10.~~.10.10/24`, gateway `10.~~.10.4`, DNS `10.~~.1.10`.
2. Create a Cisco manual binding using the PC's actual DHCP client identifier or hardware address.

The static method is simplest for the lab because `.10` is already excluded from dynamic allocation. Do not invent a client identifier; obtain it from the real client or an existing DHCP binding.

### Step 3 — Add the camera reservations from the source notes

Replace each sample identifier with the real camera identifier:

```text
configure terminal
ip dhcp pool CAMERA6
 host 10.~~.50.6 255.255.255.0
 client-identifier <camera-6-client-id>
ip dhcp pool CAMERA8
 host 10.~~.50.8 255.255.255.0
 client-identifier <camera-8-client-id>
end
```

### Step 4 — Optional external-server relay

If an external DHCP server at `10.~~.1.10` owns the scopes instead of `COREbaba`, remove the overlapping Cisco local pools and relay client VLANs to that server:

```text
configure terminal
interface Vlan10
 ip helper-address 10.~~.1.10
interface Vlan50
 ip helper-address 10.~~.1.10
interface Vlan100
 ip helper-address 10.~~.1.10
end
```

The external server needs one matching scope per VLAN, and routing/firewalls must allow the return path.

## Standalone FortiGate alternative

Use this only in a different topology where the FortiGate owns the client-facing interface. For example, if `PC-VLAN10` exists on the FortiGate at `10.~~.10.4/24`, edit that interface under **Network > Interfaces**, enable DHCP, and use a non-overlapping range such as `10.~~.10.101–199`.

This alternative is not used with the Cisco `COREbaba` gateway design. A FortiGate DHCP server is tied to its interface/subnet; it should not be presented as the local server for a VLAN whose SVI belongs to another router.

## Verification

On `COREbaba`:

```text
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict
show ip interface brief
```

On a client, release and renew the lease, then verify all values: address, `/24` mask, gateway `10.~~.10.4`, DNS `10.~~.1.10`, and lease duration.

<img width="1435" height="862" alt="1" src="https://github.com/user-attachments/assets/96b05a58-99fa-4c86-81de-1eaad95283a1" />


## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Client gets `169.254.x.x` | No offer reaches the VLAN | Check the access port, SVI state, pool network, exclusions, and bindings |
| Client gets the wrong gateway | Scope points to the wrong routing device | Use the matching `COREbaba` `.4` SVI |
| Client receives an address from another subnet | Wrong access VLAN or rogue DHCP server | Verify the port VLAN and identify the offer source |
| PC cannot receive `.10` dynamically | `.10` is excluded but no manual binding exists | Keep it static or create a binding with the real identifier |
| New clients fail after the pool fills | Range exhausted | Inspect bindings and adjust the documented exclusions deliberately |
| Relay sends requests but no offer returns | Missing external scope, route, or authorization | Verify the server's scope and both-way routing |
| Duplicate offers appear | Cisco and FortiGate DHCP are both active | Disable the server that does not own this topology |

## Notes

- Save the Cisco configuration after successful testing.
- DHCP does not require FortiGuard licensing.
- Document every static assignment and reservation to prevent conflicts.
