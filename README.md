# Fortinet Lab Documentation

FortiGate 60F lab guides using the student monitor-number addressing convention:

- Replace `~~` with the local student's monitor number.
- Standard student PC: `10.~~.10.10/24`.
- Student PC gateway: `10.~~.10.4` on Cisco `COREbaba-~~`.
- FortiGate `internal1`: `10.~~.~~.1/24`.
- `COREbaba-~~` routed FortiGate handoff: `10.~~.~~.4/24`.

## Day 1 Cisco foundation

Start with [VLAN.md](VLAN.md) to configure `COREtaas-~~`, `COREbaba-~~`, VLANs, SVIs, access ports, and the LACP trunk. Then use [DHCP.md](DHCP.md) and [OSPF.md](OSPF.md).

Guides marked **Day 1 Cisco prerequisite** depend on that switching topology and should not be started until both core switches and the FortiGate handoff are working.

## VPN and WAN labs

- [Site-to-Site-VPN.md](Site-to-Site-VPN.md) — route-based IPsec with OSPF across the tunnel.
- [SSLVPN.md](SSLVPN.md) — FortiClient SSL VPN access to the main LAN and student PC VLAN.
- [Remote-Access-IPSec-VPN.md](Remote-Access-IPSec-VPN.md) — FortiClient dial-up IPsec access.
- [SDWAN.md](SDWAN.md) — separate dual-WAN PLDT/DITO homelab with embedded screenshots.

The SD-WAN guide records a separate working topology and does not require `COREtaas-~~` or `COREbaba-~~` unless its LAN side is deliberately adapted.

## Routing and policy labs

- [Static-Routing.md](Static-Routing.md)
- [OSPF.md](OSPF.md)
- [BGP.md](BGP.md)
- [ECMP.md](ECMP.md)
- [Policy-Based-Routing.md](Policy-Based-Routing.md)
- [Firewall-Policies-and-NAT.md](Firewall-Policies-and-NAT.md)
- [Traffic-Shaping-QoS.md](Traffic-Shaping-QoS.md)

## Identity and infrastructure labs

- [LDAP-Active-Directory.md](LDAP-Active-Directory.md)
- [FSSO.md](FSSO.md)
- [RADIUS.md](RADIUS.md)
- [SNMP.md](SNMP.md)
- [Syslog.md](Syslog.md)
- [Automation-Stitches.md](Automation-Stitches.md)

## Alternative or independent designs

- [FortiLink.md](FortiLink.md) uses FortiSwitch and is separate from the Cisco collapsed-backbone design.
- [High-Availability.md](High-Availability.md) covers FortiGate HA.
- [VIP-Port-Forwarding.md](VIP-Port-Forwarding.md) covers destination NAT and port forwarding.
- [FortiGate60F-Setup.md](FortiGate60F-Setup.md) covers factory reset and initial FortiGate setup.
