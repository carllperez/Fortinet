# FortiGate 60F — FortiClient Remote-Access IPsec VPN

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Give remote FortiClient users routed access to the lab LAN |
| License | No paid FortiGuard subscription required |
| Est. time | 25–40 minutes |

## Overview

This lab builds a dial-up, route-based IPsec VPN. The FortiGate does not know the client's public address in advance, so it accepts an authenticated FortiClient on `wan1`, assigns an address from a private pool, and installs split routes for the lab LAN.

This differs from `SSLVPN.md`: IPsec uses IKE/IPsec (normally UDP 500 and 4500), a pre-shared key, and phase 1/phase 2 negotiation. SSL VPN uses TLS on the configured SSL-VPN port and an SSL-VPN portal. Both can use the same local users, but their tunnel objects, client profiles, and firewall policies are separate.

> Replace `~~` with the student's monitor number throughout. Example: monitor 10 gives student PC `10.10.10.10` and protected network `10.10.10.0/24`.
>
> **Screenshot note:** The screenshots in this guide were captured by student/monitor 10. Therefore, the GUI displays `10.10.10.0/24` and a VPN client pool beginning with `10.10.250`. The object name `LAN-~~` intentionally retains the reusable placeholder.

## Prerequisites

- FortiClient VPN installed on a computer outside the FortiGate LAN
- A reachable public or lab-routed address on `wan1`
- No upstream carrier-grade NAT unless UDP 500/4500 can be forwarded
- Protected network `10.~~.10.0/24` on `dmz`
- A configuration backup

## Network overview

| Item | Value |
|---|---|
| FortiGate WAN | `NET1 (wan1)`, using the assigned upstream address |
| Protected LAN | `dmz`, `10.~~.10.0/24` |
| VPN name | `RA-IPsec` |
| Local user | `ipsec-user` |
| User group | `IPsec-Users` |
| Client pool | `10.~~.250.10–10.~~.250.50` |
| IKE | IKEv2 where the installed FortiClient supports it |
| Tunnel mode | Split tunnel to `10.~~.10.0/24` |

This standalone version protects the network directly connected to `dmz`. It does not require `COREtaas` or `COREbaba`. The standard student PC is `10.~~.10.10`, and the FortiGate DMZ address is normally `10.~~.10.1/24`.

```text
Remote FortiClient ── Internet/lab WAN ── wan1 [ FortiGate ] dmz ── Lab PC
                                                                  10.~~.10.10
```

## Configuration

### Step 1 — Create the user and group

1. Go to **User & Authentication > User Definition** and create a **Local User**.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/d657214b-804f-4160-94ec-fbe1893c9fe9" />

2. Use a strong lab password and name the user `ipsec-user`.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/553f554d-90f8-42ba-ba72-5a8a92b8ea6d" />

3. Leave two-factor authentication disabled for this basic lab unless a FortiToken has been assigned.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/62922958-3f6a-4fe8-b99e-5790b877eb66" />

   Confirm that the account is enabled, then submit it.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/a43c99e0-9ffb-4a97-9cbe-3f234c2ecc76" />

4. Go to **User & Authentication > User Groups**.

   The page may initially show only the default or previously created groups.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/bf73a2bb-46c6-4b42-b616-85c927f3aca0" />

5. Create a Firewall group named `IPsec-Users` and add `ipsec-user`.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/6b18da4e-dac8-4b0a-bafc-d922791dfa8b" />

### Step 2 — Create the protected-network address

Go to **Policy & Objects > Addresses** and create:

| Field | Value |
|---|---|
| Name | `LAN-~~` |
| Type | Subnet |
| Subnet | `10.~~.10.0/255.255.255.0` |
| Interface | `any` |

The monitor 10 example appears as follows:

<img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/2ee192c6-989e-48e6-af2b-d72d5e99a24d" />

### Step 3 — Run the IPsec wizard

1. Go to **VPN > IPsec Wizard**.

   The wizard may initially open with **Site to Site** selected. This is only the starting screen; change the template in the next step.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/e68f362f-04e5-46a0-bba0-02babf68ce07" />

2. Name the tunnel `RA-IPsec`.
3. Choose **Remote Access** and **FortiClient**.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/5fafdd0a-4673-475c-be9a-9d88def0b72c" />

4. Set Incoming Interface to `NET1 (wan1)`.
5. Select pre-shared-key authentication, enter a long random key, and select `IPsec-Users`.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/a42b87dc-f8ff-47eb-833b-16872d6f11c3" />

   > **Security note:** The screenshot contains a visible classroom example key. Do not reuse that value. Replace it with a unique pre-shared key before using or publishing a live configuration.

6. Set Local Interface to `dmz` and Local Address to `LAN-~~`.
7. Set the client range to `10.~~.250.10–10.~~.250.50` with mask `255.255.255.255` if offered by the wizard.
8. Enable IPv4 split tunnel and keep `LAN-~~` as the protected destination.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/6464dbbd-7b0f-4966-bfe2-16b9d1acffcf" />

9. Under Client Options, enable **Save Password** only if permitted by the lab. Leave **Auto Connect** and **Always Up** disabled for a manual lab connection.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/8d63fdd3-f528-47d7-816a-ef72dd8fc5e7" />

10. Select IKEv2 if the FortiOS 7.0 wizard and installed FortiClient expose the same option. If not, keep the wizard's compatible default; both ends must use the same IKE version.
11. Review the generated tunnel, address pool, route, and firewall policy before selecting **Create**.

   <img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/0e842365-b2ae-449b-b44d-f1d98e43ea1f" />

> Gotcha: a phase 1 pre-shared key proves that the endpoint knows the shared secret; the username and password identify the individual user. A mismatch in either stage prevents the connection.

### Step 4 — Review phase 1 and phase 2

Go to **VPN > IPsec Tunnels**, edit `RA-IPsec`, and confirm:

- Remote Gateway is Dialup User.
- Interface is `wan1`.
- The IKE version matches FortiClient.
- NAT traversal is enabled when the client may be behind NAT.
- User authentication references `IPsec-Users`.
- Mode configuration assigns the client pool.
- Phase 2 covers the protected LAN and client address space created by the wizard.

Do not casually change proposals after exporting or building the FortiClient profile. Phase 1 and phase 2 proposals must overlap on both ends.

The tunnel can show **Inactive** until a remote FortiClient successfully connects. This is normal immediately after creation.

<img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/cec0b8fc-a614-4f17-832b-82c6b714bcdc" />

### Step 5 — Review the firewall policy

The wizard should create a policy similar to:

| Field | Value |
|---|---|
| Incoming Interface | `RA-IPsec` |
| Outgoing Interface | `dmz` |
| Source | generated client-pool object `RA-IPsec_range` |
| Destination | `LAN-~~` |
| Schedule | `always` |
| Service | `ALL` for the first test |
| Action | Accept |
| NAT | Off |
| Logging | All Sessions |

NAT should be off for this VPN-to-DMZ policy. The DMZ network is directly connected to the FortiGate, so return traffic for the VPN client pool is routed through the same FortiGate. Enabling NAT would hide the real VPN client address from the lab PC.

The following real-world classroom screenshots show the wizard-generated policy with NAT enabled. This can work as a lab shortcut, but it does not preserve the original VPN client address. For the routed configuration described in the table above, turn NAT off.

<img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/b28baf7d-3fb1-4c2f-a0b9-92da9eacea0c" />

<img width="1440" height="867" alt="clipboard" src="https://github.com/user-attachments/assets/5632ca5d-c7c5-4d86-b14f-c3660e1467e5" />

### Step 6 — Configure FortiClient

In FortiClient, open **Remote Access**, add an IPsec VPN, and use:

| Field | Value |
|---|---|
| Remote Gateway | Reachable address assigned to `NET1 (wan1)` |
| Authentication Method | Pre-shared Key |
| Pre-shared Key | Same key as phase 1 |
| IKE version/proposals | Match the FortiGate |
| Username | `ipsec-user` |

If the FortiGate tunnel uses a specific peer ID, enter that value as FortiClient's Local ID. Save, disconnect the test computer from the lab LAN, and connect from a different routed network.

## Verification

1. Confirm the client receives an address from `10.~~.250.10–50`.
2. Confirm its route table contains `10.~~.10.0/24` through the VPN while its default route remains local.
3. Ping a reachable LAN host, not only the FortiGate itself.
4. In **Log & Report > VPN Events**, check the successful user and tunnel name.

```shell
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list
get router info routing-table all
```

The tunnel should have an active IKE gateway, an up selector, an assigned client IP, and increasing packet counters.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No response from gateway | WAN address is unreachable, UDP 500/4500 is blocked, or upstream NAT is missing forwarding | Test from a genuinely remote network and verify the upstream path and NAT-T |
| IKE negotiation fails immediately | Pre-shared key, IKE version, proposal, or peer ID mismatch | Compare phase 1 and FortiClient settings exactly |
| Username is repeatedly rejected | User is not in `IPsec-Users`, password is wrong, or user authentication method differs | Test the local account, group membership, and generated phase 1 settings |
| Tunnel connects but no client address appears | Mode configuration/client pool is missing or overlaps another subnet | Restore the generated pool and use a unique subnet |
| Client reaches the FortiGate but not LAN hosts | Missing tunnel-to-LAN policy, wrong policy destination, host firewall, or no return route | Check policy counters, test a known host, and route the client pool back through the FortiGate |
| Client internet stops after connection | Split tunneling is disabled or a default route was pushed | Limit the protected network to `LAN-~~` and reconnect |
| Tunnel is up but packets do not increase | Client route points elsewhere or phase 2 selectors do not cover the traffic | Inspect the client route table and both phase 2 selectors |

For IKE troubleshooting during one controlled connection attempt:

```shell
diagnose debug reset
diagnose vpn ike log-filter clear
diagnose debug application ike -1
diagnose debug enable
```

Stop immediately after the test:

```shell
diagnose debug disable
diagnose debug reset
```

## Notes

- FortiGate-generated pings are local-out traffic and do not prove that the tunnel-to-LAN firewall policy works.
- A paid FortiGuard subscription is not required for local-user dial-up IPsec. FortiClient VPN-only functionality is sufficient for this lab.
- Use unique credentials and certificates in production; a shared PSK is intentionally simple for a beginner lab.
