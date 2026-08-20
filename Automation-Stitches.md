# FortiGate 60F — Local Automation Stitches

| | |
|---|---|
| Device | FortiGate 60F |
| Firmware | FortiOS 7.0.x |
| Purpose | Respond to local interface and configuration events |
| License | No paid FortiGuard subscription required for these local examples |
| Est. time | 25–40 minutes |

## Overview

An automation stitch connects a trigger to one or more actions. The trigger is the event; the action is what FortiGate does. Stitches can run on a standalone FortiGate even when it is not part of a Security Fabric.

This lab records and tests two safe local examples:

- An interface-down event using the built-in/local Network Down or FortiOS Event Log trigger.
- A Configuration Change trigger with a harmless CLI information command for demonstration.

## Prerequisites

- Administrator access to **Security Fabric > Automation**
- Event logging enabled
- One unused lab interface that can be disconnected safely
- A configuration backup before testing any CLI Script action

## Configuration

### Step 1 — Review existing stitches

Go to **Security Fabric > Automation**. FortiOS 7.0 commonly includes default stitches such as Network Down, Reboot, HA Failover, or License Expiry Notification. Review them before creating duplicates.


### Step 2 — Create an interface-down trigger

1. Click **Create New**.
2. Create a trigger named `Lab-Interface-Down`.
3. Select the built-in Network Down trigger if present. Otherwise select **FortiOS Event Log** and choose the interface-link-status event from the GUI's event list.
4. Add a field filter for the unused test interface when the GUI exposes interface fields.

Do not hardcode a log ID copied from another firmware build. Selecting the event by description in the FortiOS 7.0 GUI keeps the trigger aligned with that device's log schema.

<img width="1440" height="867" alt="1" src="https://github.com/user-attachments/assets/6a150ac1-8141-4941-bdfb-1a78384d9ee1" />




### Step 3 — Add a local action

Add an action supported by the local appliance, such as an alert/notification or CLI Script. Cloud functions, FortiGate Cloud event handlers, and third-party webhooks are optional and may require integration.

For a CLI Script action, use a read-only command for the first test:

```shell
get system status
```

Name the action `Record-Lab-Status`, attach it to the stitch, and enable the stitch.

<img width="1440" height="870" alt="2" src="https://github.com/user-attachments/assets/830fa740-570d-485b-a7ab-b89ea09c1026" />
<img width="1440" height="866" alt="3" src="https://github.com/user-attachments/assets/61778b3f-0455-441a-bea4-c3a2b5467d6c" />
<img width="1440" height="869" alt="4" src="https://github.com/user-attachments/assets/aa093be6-58f3-484b-9816-5b08f7cb0a96" />
<img width="1440" height="870" alt="5" src="https://github.com/user-attachments/assets/49600397-a34e-47e8-bcdf-59c8b3140090" />


Automation activation and action results are recorded in FortiGate event/automation history. There is no need to invent a separate generic "write log" action when the selected 7.0 GUI does not offer it.

### Step 4 — Create a configuration-change example

Create another stitch with the **Configuration Change** trigger. For the first lab, attach only a local notification or the same read-only CLI Script action. Do not attach a script that reverts all configuration or creates new objects.

> Gotcha: a CLI Script action runs with appliance privileges and without an interactive reviewer. Treat it like configuration automation, not a convenient scratchpad.

<img width="1440" height="869" alt="6" src="https://github.com/user-attachments/assets/3accb7c2-c049-49d8-b0be-df91b9036d71" />
<img width="1440" height="870" alt="7" src="https://github.com/user-attachments/assets/55962750-f124-429e-9ac9-5dd7d40ab33d" />


## Verification

Test a stitch without waiting for the event:

```shell
diagnose automation test Lab-Interface-Down
diagnose test application autod 2
diagnose test application autod 3
```

Then perform a real test:

1. Disconnect only the unused interface selected by the trigger.
2. Open **Log & Report > System Events** and the Automation page/history.
3. Confirm the trigger time, stitch name, and action result.
4. Reconnect the interface.
5. Edit the comment on a disposable address object and confirm the Configuration Change stitch activates once.

`diagnose test application autod 2` displays configured stitches; option `3` displays activation statistics and last-trigger information.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Test command says stitch not found | Name/case differs or stitch was not saved | Copy the exact stitch name from the GUI |
| Manual test works but cable pull does not | Wrong event, interface filter, or interface never changed operational state | Check the actual system event and rebuild the trigger from that GUI event |
| Trigger fires but action fails | CLI command is unsupported in automation context or action is disabled | Use a simple read-only first action and inspect automation history |
| Configuration change causes repeated actions | The action itself changes configuration and retriggers the stitch | Remove configuration-writing actions from that trigger |
| Cloud/webhook action fails | Internet, DNS, certificate, credential, or service dependency is missing | Use local actions for this subscription-free lab |
| No history appears | Event logging or the stitch is disabled | Enable both and perform a fresh event |

## Notes

- Test on an unused interface. Pulling the management or only WAN link can disconnect the GUI.
- Automation stitches reduce response time but can amplify a mistake quickly. Keep scripts small, idempotent, and backed up.
- The local triggers and CLI action used here do not require FortiGuard services.
