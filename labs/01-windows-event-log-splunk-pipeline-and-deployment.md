# Lab 01: Build, Troubleshoot, and Redeploy a Windows Event Log to Splunk Pipeline with Dashboards and Alerts

## Objective

Configure a Splunk Universal Forwarder to ship Windows Security, System, and Application event log data to a Splunk indexer, generate realistic test security events, and diagnose a real end-to-end pipeline failure down to its actual root cause. Then rebuild the Splunk indexer from scratch on a fresh Ubuntu VM in Azure, scope its network security rules to trusted sources only, reconfigure the Universal Forwarder against the new indexer, validate the pipeline with test data, and build a security-monitoring dashboard and a scheduled alert for high-privileged logon activity.

---

## Scenario

A Windows VM's Universal Forwarder is configured to monitor the Security, System, and Application event logs and route them to a dedicated `windows_logs` index. A PowerShell script generates realistic test events (failed logons, a successful logon, service restarts, application log entries, an account lockout) to validate the pipeline. After running the script and waiting the recommended interval, `index="windows_logs" | head 100` returns nothing. The pipeline needs to be diagnosed and fixed without assuming where the fault lies.

Once that incident is resolved, the indexer itself needs to be rebuilt from scratch on Ubuntu so it can be managed remotely from the terminal, with network access locked down before anything else is configured. After the new indexer is reachable and receiving is enabled, the Windows server's Universal Forwarder is reinstalled and pointed at it, forwarding the same three event logs into the same dedicated index. Once real data is confirmed flowing again, that data is turned into something usable: a dashboard for at-a-glance visibility into login activity, and a scheduled alert that fires automatically on high-privileged logon activity instead of relying on someone to go looking for it.

---

## Environment

| Component | Description |
| --- | --- |
| Platform | Microsoft Azure |
| Forwarder | Splunk Universal Forwarder, Windows VM |
| Original Indexer | Splunk Enterprise, Linux VM (Azure), same `10.0.0.0/24` subnet as the forwarder |
| Rebuilt Indexer | Splunk Enterprise, Ubuntu VM (Azure Spot instance, 1 vCPU / 8 GB memory), new dedicated resource group |
| Target Index | `windows_logs` |
| Splunk Web Port | 8000 (restricted to admin IP, rebuilt indexer) |
| SSH Port | 22 (restricted to admin IP, rebuilt indexer) |
| Forwarding Port | 9997 (restricted to VNet range `10.0.0.0/24`) |
| Primary Focus | Log source configuration, audit policy, network path validation, Splunk config-loading diagnostics, least-privilege network design, dashboards, and alerting |

---

## Test Event Generation Script

A PowerShell script (`scripts/Windows-SecurityEvent-Generator.ps1`) generates test activity across five categories, and is reused later in this lab to validate the rebuilt pipeline:

- **Failed logons (15x):** NTLM authentication attempts against the local SAM via `net use \\localhost\IPC$ /user:.\<testuser> <wrongpassword>`, generating Event ID 4625
- **Successful logon (1x):** the same technique with the correct password, generating Event ID 4624
- **Service restarts (3x):** stopping and restarting the Spooler, Task Scheduler, and Netlogon services, generating Event ID 7036
- **Application log entries (5x):** custom events written to a dedicated `SplunkLabTest` event source, Event ID 1001
- **Account lockout (20x):** repeated failed NTLM attempts to trigger Event ID 4740

The script originally used `Start-Process -Credential` to simulate logons, which is unreliable outside a fully interactive desktop session (it fails with "the stub received bad data" over RDP-without-session, remoting, or SSH). It was rewritten to use `net use` against the loopback `IPC$` share instead, which forces a real NTLM authentication attempt against the local SAM and works reliably from any session type. This does change the recorded logon type from Interactive to Network, which is worth knowing if a downstream search or dashboard filters on logon type specifically.

---

## Steps Performed

### Part 1: Build and Troubleshoot the Original Pipeline

1. Configured `inputs.conf` on the forwarder with `[WinEventLog://Security]`, `[WinEventLog://System]`, and `[WinEventLog://Application]` stanzas, each set to `disabled = 0` and `index = windows_logs`.
2. Ran the test event generation script and waited 60 seconds, then searched `index="windows_logs" | head 100` in Splunk. Zero results.
3. Checked the target index directly with `eventcount summarize=false index=*` — confirmed `windows_logs` existed but had 0 events, and noticed a separate `splunklogger` index was disabled. This initially looked like a routing mismatch, but turned out to be an unrelated leftover from earlier configuration, not the current issue.
4. Reviewed `inputs.conf` directly and confirmed all three `WinEventLog` stanzas were correctly set to `index = windows_logs` with no `disabled = 1` anywhere — ruling out the obvious index-mismatch explanation.
5. Confirmed Windows Advanced Audit Policy was correctly configured via `auditpol /get /category:"Logon/Logoff"` and `auditpol /get /category:"Account Logon"`: Logon and Account Lockout were both set to "Success and Failure," and Credential Validation was "Success and Failure" — ruling out audit policy as the cause.
6. Independently verified Windows was capable of generating and logging real events: captured the Security log's record count before and after a single manual `net use` failed-logon attempt (54,234 → 54,238), and confirmed the expected `System error 1326: The user name or password is incorrect` response — proving Windows event generation itself was working correctly.
7. Checked whether the forwarder was actually connected to the indexer. `Test-NetConnection -ComputerName localhost -Port 8089` (the forwarder's own local management port) failed on both `::1` and `127.0.0.1`, even though the `splunkd` process was confirmed running via `Get-Process`.
8. Reviewed the forwarder's `splunkd.log` and found repeated `AutoLoadBalancedConnectionStrategy — Connected to idx=10.0.0.5:9997` entries roughly every 30 seconds — confirming the forwarder was in fact successfully connecting to the indexer on the data-forwarding port (this cycling is normal `autoLBFrequency` behavior, not a symptom of a problem), which ruled out basic network connectivity as the cause.
9. Reviewed the Azure NSG rules on the indexer's network interface and confirmed a rule ("Vnet_Logs," priority 1030) explicitly allowing inbound TCP 9997 from the `10.0.0.0/24` subnet, and confirmed the forwarder's own IP fell within that range.
10. Confirmed on the indexer that Splunk's receiving configuration had port 9997 actively listening (Settings > Forwarding and receiving).
11. With audit policy, event generation, the network path, and the receiving configuration all independently confirmed correct, ran `splunk btool inputs list --debug` on the forwarder to see the actual merged, effective configuration Splunk was running — and found no `WinEventLog://Security`, `WinEventLog://System`, or `WinEventLog://Application` stanzas anywhere in the output, despite them clearly existing in the `inputs.conf` file reviewed in step 4.
12. Ran `Get-ChildItem` on the forwarder's `system\local` config directory and found the actual filename was `input.conf`, not `inputs.conf`. Splunk only loads files matching the exact expected filename, silently ignoring anything else, with no error or warning logged anywhere.
13. Renamed the file to `inputs.conf`, restarted the forwarder service, reran the test event generation script, and confirmed events began appearing in `index="windows_logs"`.

### Part 2: Rebuild the Indexer, Redeploy the Forwarder, and Build Dashboards and Alerts

14. Deleted the VM that had hosted the original indexer during Part 1 and created a new Azure virtual machine in a new, dedicated resource group ("Splunk lab") to keep the rebuild isolated from the earlier troubleshooting work.
15. Named the VM clearly (VM2 Splunk) and selected Ubuntu as the OS so Splunk could be installed and managed entirely from the terminal.
16. Sized the VM using the lab's standard choices: Spot pricing, 1 vCPU, 8 GB memory.
17. On the first pass through the Networking tab, left the NSG on its Basic default (SSH allowed from all IP addresses, with Azure's own warning that this is only recommended for testing) and a newly generated, non-default subnet — caught both before deploying.
18. Switched to an Advanced NSG (`VM2_Splunk`), reverted the subnet back to the existing default `10.0.0.0/24` range, and scoped rules to least privilege: port 8000 (Splunk Web) and port 22 (SSH) restricted to my own IP address only, and port 9997 (forwarding) restricted to the VNet range `10.0.0.0/24` rather than exposed publicly, since only in-network forwarders need to reach it. Verified rule names and priorities before deployment, then reviewed the VM configuration end to end and deployed it, recording its public IP address for remote access.
19. Connected to the Ubuntu VM using PuTTY with the recorded public IP and confirmed terminal access before installing anything.
20. Downloaded the Splunk Linux installation package (`splunk-10.4.2-33c3bf42cd73-linux-amd64.deb`, 1.2 GB) via `wget` to the VM. The first two `sudo dpkg -i` attempts failed against filenames that didn't match what was actually on disk (`splunk-10.2.2-linux-amd64.deb`, then `splunk-10.4.2-linux-amd64.deb` — missing the build-hash segment of the real filename); running `ls` to confirm the exact filename and re-running against it succeeded.
21. Started the Splunk service and enabled boot-start so it would relaunch automatically after a VM restart, confirming the service started without errors.
22. Opened Splunk Web on the VM's public IP over port 8000, signed in with the admin account, and confirmed the NSG rule for 8000 was working as intended.
23. Under Settings > Forwarding and receiving, added a new receiving port on 9997 and saved the configuration; under Settings > Indexes, created a new `windows_logs` index so forwarded events had a destination.
24. On the Windows Server VM, downloaded and installed the Splunk Universal Forwarder, leaving the deployment server field blank and entering the Splunk VM's private IP and port 9997 as the receiving indexer.
25. Located the Universal Forwarder's local configuration directory and wrote `[WinEventLog://Security]`, `[WinEventLog://System]`, and `[WinEventLog://Application]` stanzas, each routed to the `windows_logs` index (see `scripts/universal-forwarder-inputs.conf`). The file was initially saved as `input.conf` — the exact same one-character naming mistake made and fixed earlier in this lab (step 12) — before being corrected to `inputs.conf`; since Splunk only loads a file with that exact name, this had to be fixed for the pipeline to work at all.
26. Restarted the Universal Forwarder service so the new configuration took effect, then ran the test event generation script to produce sample login activity and confirmed it created and cleaned up its own temporary test user, waiting roughly 60 seconds for the logs to reach Splunk.
27. In Splunk Search, set the time range to All time, searched the `windows_logs` index, and confirmed events were arriving — validating with searches for successful logins and after-hours logins.
28. Created a new Classic dashboard named "Windows Security Overview" (private, no description) with panels for account activity over the last 24 hours, top processes over the last 24 hours, login activity over time, and after-hours logins, then saved it and added it to the home page.
29. Built and validated a scheduled alert: ran the alert query in Search, saved it as an alert named "High-Privileged Logon Count," set it to run every 15 minutes (`*/15 * * * *`), configured it to trigger when the number of results is greater than zero, set the action to add the event to Triggered Alerts, and confirmed it appeared in the alert list.

---

## Root Cause (Part 1 Incident)

A configuration file at `system\local\input.conf` (missing the "s") was never loaded by Splunk, because Splunk only reads files that exactly match `inputs.conf`. Every other component in the pipeline, Windows audit policy, event generation, the Azure NSG rule, the Linux indexer's receiving configuration, and the TCP connection between forwarder and indexer, was correctly configured the entire time.

---

## Troubleshooting Checklist Used (Part 1 Incident)

| Layer | Check | Result |
| --- | --- | --- |
| Target index | `eventcount summarize=false index=*` | Index exists, 0 events (inconclusive on its own) |
| Config file content | Reviewed `inputs.conf` directly | Content looked correct — misleading, since this wasn't the file actually loaded |
| Windows audit policy | `auditpol /get /category:...` | Correctly configured (Logon, Account Lockout, Credential Validation = Success and Failure) |
| Event generation | `Get-WinEvent -ListLog Security` record count before/after a manual test | Confirmed Windows generating real events |
| Local forwarder health | `Test-NetConnection -ComputerName localhost -Port 8089` | Management port unreachable, but `splunkd` process was running |
| Forwarder → indexer connectivity | `splunkd.log` on the forwarder | Confirmed repeated successful connections to the indexer on 9997 |
| Cloud network rule | Azure NSG on the indexer's NIC | Confirmed 9997 allowed from the forwarder's subnet |
| Splunk receiving config | Settings > Forwarding and receiving | Confirmed listening on 9997 |
| **Effective, merged config** | `splunk btool inputs list --debug` | **Confirmed the WinEventLog stanzas were never actually loaded** |
| Root cause | `Get-ChildItem` on the config directory | File was named `input.conf`, not `inputs.conf` |

---

## Network Security Group Rules (Rebuilt Indexer)

| Port | Purpose | Source Restriction |
| --- | --- | --- |
| 8000 | Splunk Web | Admin IP only |
| 22 | SSH | Admin IP only |
| 9997 | Forwarder → Indexer (data) | VNet range `10.0.0.0/24` only — never exposed publicly |

The VM's networking blade defaults to a Basic NSG with SSH open to all IP addresses and a freshly generated subnet — both were caught and corrected to the table above before the VM was deployed, rather than after (step 17-18).

---

## Dashboard: Windows Security Overview

| Panel | Purpose |
| --- | --- |
| Account activity (last 24h) | Recent account-related event volume at a glance |
| Top processes (last 24h) | Surface unusual or high-frequency process activity |
| Login activity over time | Trend view of logon volume for anomaly spotting |
| After-hours logins | Isolate logons occurring outside expected working hours |

---

## Alert: High-Privileged Logon Count

| Setting | Value |
| --- | --- |
| Type | Scheduled |
| Schedule | Every 15 minutes (`*/15 * * * *`) |
| Trigger Condition | Number of results greater than 0 |
| Action | Add to Triggered Alerts |

This alert automates detection of privileged-account logon activity instead of relying on someone manually searching for it, closing the loop between "the data is in Splunk" and "someone gets notified when it matters."

---

## Security Concepts Practiced

- Validating audit policy before assuming a log source itself is broken, since Windows won't write events Splunk was never going to see in the first place
- Independently verifying that a monitored system is capable of generating the events being tested for, rather than trusting a script's own "success" output
- Distinguishing "the network path is open" from "the application is using it correctly," since a healthy TCP connection doesn't guarantee the right data is flowing over it
- Using a tool's own effective-configuration output (`btool`) as a source of truth over reading a single config file directly, since file content on disk doesn't guarantee that file is actually being used
- Documenting a real troubleshooting path, including two lines of investigation that turned out to be red herrings (the `splunklogger` index, the local management port), rather than only presenting the final answer
- Scoping network access by least privilege before installing or configuring anything: admin-only access to management interfaces (8000, 22), and VNet-only access to the data-forwarding port (9997) rather than exposing it publicly
- Building the rebuilt indexer in a fresh, isolated resource group rather than reusing prior infrastructure, avoiding cross-contamination between environments
- Validating that data is actually flowing into an index before building visualizations on top of it, instead of assuming the pipeline works
- Translating raw log data into operational visibility (a dashboard) and automated detection (a scheduled alert) rather than treating ingestion as the end goal
- Using a disposable test account and script-generated events to validate the pipeline without touching real user activity

---

## Evidence

Part 1 (the original troubleshooting incident) is documented directly in this writeup via terminal output, Windows Event Log record counts, and `splunkd.log` / `btool` excerpts, rather than screenshots. The test event generation script is included at:

```text
/scripts/Windows-SecurityEvent-Generator.ps1
```

Part 2 (indexer rebuild, forwarder redeployment, dashboard, and alert) includes screenshot evidence, at:

```text
/screenshots/lab-01-splunk-pipeline-and-deployment/
```

The Universal Forwarder input configuration used in Part 2 is included at:

```text
/scripts/universal-forwarder-inputs.conf
```

Reference video: [Deploy and Configure Splunk for Windows Event Log Monitoring in Azure](https://loom.com/share/4e2bb427a6794b88b079367dcf052ab7)

---

## Outcome

A Windows Event Log to Splunk forwarding pipeline was built, tested with a custom PowerShell event-generation script, and — after appearing to deliver zero events despite every visible configuration looking correct — was diagnosed layer by layer until the actual root cause was found: a configuration file that Splunk was silently never loading due to a one-character filename mismatch. After the fix, events began flowing into the target index as expected.

The indexer was then rebuilt from scratch on Ubuntu in an isolated Azure resource group, with network access scoped to least privilege from the start. The Universal Forwarder was reinstalled and pointed at the new indexer, Windows Event Log data was successfully forwarded into the same `windows_logs` index, validated with test events, and turned into an operational "Windows Security Overview" dashboard and a scheduled "High-Privileged Logon Count" alert that checks for new high-privilege logon activity every 15 minutes.

---

## Lessons Learned

The most important discipline in the first half of this lab was refusing to stop at "the config file looks right" and instead checking what Splunk was actually running with `btool`. Every individual layer, audit policy, event generation, the NSG rule, the Linux firewall path, and the receiving configuration, tested out correctly on its own, which made it easy to assume the problem had to be somewhere still unchecked in that same list. The actual fault was invisible to every one of those checks, because the file defining the inputs was never being read by Splunk at all.

Two side investigations turned out to be dead ends, and are documented rather than edited out: a disabled `splunklogger` index initially looked like a routing mismatch but was an unrelated leftover from earlier configuration, and a failed local `Test-NetConnection` to the forwarder's management port (8089) raised concern about the forwarder's health but turned out to be unrelated to the actual data-forwarding path (which uses port 9997, not 8089).

The real lesson from Part 1: when every individual component checks out but the end-to-end result still fails, the next step should be checking whether the system is actually using the configuration you're looking at, not re-checking the same components more carefully. `btool inputs list --debug` answered in one command what a dozen individual checks couldn't, since it showed the true, merged configuration rather than a single file's contents.

Scoping the network security group before touching Splunk itself in Part 2 — admin-only access to 8000 and 22, VNet-only access to 9997 — meant the rebuilt environment was never briefly exposed while still being configured, though it took a second pass through the Networking tab to get there: the default options (Basic NSG, SSH open to all IPs, a brand-new subnet) had to be caught and corrected before deploying, not after. Confirming data was actually flowing into the index before building the dashboard avoided the wasted effort of building panels against an empty data source and then having to debug whether the dashboard or the pipeline was at fault. The dpkg install step needed two corrections before the package filename matched what was actually downloaded — a small mismatch there fails immediately and obviously, which is a much easier problem to catch than the silent config-loading issue from Part 1.

Most notably, the exact `input.conf` vs `inputs.conf` naming mistake from Part 1 (step 12) happened again in Part 2 (step 25), on a completely rebuilt environment — proof that "I already learned this lesson" doesn't prevent the same typo from recurring, and that double-checking a config filename immediately after saving it is a habit worth building, not just a one-time fix. Restarting the Universal Forwarder service immediately after any configuration change, rather than troubleshooting against a stale config, avoided repeating Part 1's other mistake of debugging a problem that a simple restart would have already fixed.
