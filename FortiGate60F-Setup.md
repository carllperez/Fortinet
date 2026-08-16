# FortiGate 60F — Factory Reset and Initial Lab Setup

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.15 |
| Purpose | Build a clean FortiGate |
| Management | SecureCRT serial console and FortiGate web interface |
| Est. time | 10-20 minutes |

## Overview

This guide resets a FortiGate 60F to factory defaults and prepares it for a structured security lab. The serial console is kept connected throughout the reset so management access is available even while the network configuration is changing.

The intended final topology is:

```text
External/VPN PC                 FortiGate 60F                 Internal client
10.10.10.2/24 ---------------> wan1                      internal1 ---------->
                                10.10.10.1/24               192.168.10.1/24
```

> On a FortiGate 60F, use the dedicated `wan1` interface for the simulated external network. The front-panel port labeled **1** is represented in FortiOS as `internal1`; it is not named `port1`.

## Prerequisites

- FortiGate 60F running FortiOS 7.0.15
- SecureCRT connected to the FortiGate serial console
- Working administrator credentials for the existing configuration
- Access to the FortiGate web interface
- A saved copy of any configuration that may be needed later
- Serial settings: 9600 baud, 8 data bits, no parity, 1 stop bit, and no flow control

## Proposed network overview

| Item | Value |
|---|---|
| External interface | wan1 |
| External interface IP | 10.10.10.1/24 |
| External test PC | 10.10.10.2/24 |
| Internal interface | internal1 |
| Internal gateway | 192.168.10.1/24 |
| FortiGate firmware | FortiOS 7.0.15 |

## Configuration

### Step 1 — Back up the existing configuration

Even when the old configuration will not be reused, save a backup before resetting the appliance.

1. Log in to the FortiGate web interface.
2. Open the administrator menu in the upper-right corner.
3. Select **Configuration > Backup**.
4. Select **Local PC** as the destination.
5. Save the configuration using a descriptive filename.
6. Store the backup password if encryption is enabled.

From SecureCRT, record the current model, serial number, firmware build, operating mode, and system time:

```shell
get system status
```

> A configuration backup provides a recovery path if an interface, license-related setting, certificate, policy, or VPN configuration is needed later.

### Step 2 — Reset the FortiGate to factory defaults

Keep SecureCRT connected to the serial console. Run:

```shell
execute factoryreset
```

FortiGate displays a warning similar to:

```text
This operation will reset the system to factory default!
Do you want to continue? (y/n)
```

Enter:

```text
y
```

The FortiGate erases the active configuration and reboots. Do not disconnect power while it is restarting.

> `execute factoryreset` resets the configuration but does not downgrade the installed FortiOS firmware or reset the installed antivirus and IPS definition versions.

### Step 3 — Complete the first console login

1. Wait for the FortiGate login prompt to return.
2. Log in with the default administrator account:

```text
Login: admin
Password: leave blank
```

3. When prompted, create and confirm a strong administrator password.
4. Confirm the appliance is operational:

```shell
get system status
```

Expected results include:

- Model: FortiGate 60F
- Firmware: FortiOS 7.0.15
- Status: operating normally
- Current administrator session connected through the console

## Verification

After the reset and first login, verify the factory-default interface configuration:

```shell
show system interface
```


## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No console output | Incorrect serial settings or cable | Use 9600 baud, 8-N-1, no flow control, and verify the correct COM port |
| Reset command is rejected | Insufficient administrator permissions | Log in with a `super_admin` account |
| FortiGate does not return to the login prompt | Appliance is still rebooting | Wait several minutes and keep the serial session connected |
| `admin` with a blank password fails | Factory reset did not complete or another default process applies | Review the console boot output and confirm the reset completed |
| Web interface is unavailable after reset | PC is connected to the wrong interface or subnet | Use the serial console to inspect `show system interface` before changing the PC address |

## Notes

- The serial console is the primary recovery method during the initial build.
- Do not expose administrative HTTPS or SSH on `wan1` in a production deployment.
- The external test PC at `10.10.10.2` will later simulate a remote Internet user.
- `wan1` will be used for remote-access VPN and future ADVPN exercises.
- `internal1` will be used for firewall-policy and web-filtering exercises.
- The guide should be updated after each successfully verified configuration stage.

## References

- [FortiOS 7.0.15 configuration backups and reset](https://docs.fortinet.com/document/fortigate/7.0.15/administration-guide/702257/configuration-backups-and-reset)
- [FortiGate 60F and 61F interface architecture](https://docs.fortinet.com/document/fortigate/7.2.0/hardware-acceleration/758378/fortigate-60f-and-61f-fast-path-architecture)
