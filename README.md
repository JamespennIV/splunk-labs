# Splunk SIEM Log Pipeline Lab

## Project Overview

This project documents building a Windows Event Log → Splunk forwarding pipeline for SIEM ingestion, in two parts: diagnosing and resolving a real end-to-end "zero events indexed" incident that touched every layer of the pipeline (Lab 01), and rebuilding the Splunk indexer from scratch on Ubuntu with least-privilege network rules, a validated forwarding pipeline, and an operational dashboard and scheduled alert (Lab 02).

The lab simulates the exact kind of work a SOC analyst or security engineer does with a SIEM: standing up ingestion correctly, troubleshooting it when it silently breaks, and turning raw log data into dashboards and alerts that actually get looked at.

---

## Business Scenario

A Windows host needs to ship Security, System, and Application event log data to a Splunk indexer so it can be searched and alerted on. The pipeline needs to demonstrate:

- Realistic test event generation for common security scenarios (failed/successful logons, account lockout, service restarts)
- Correct Windows audit policy so those events are actually written to the log in the first place
- A Universal Forwarder correctly configured to monitor and ship the right event logs to the right index
- A validated network path from forwarder to indexer, including cloud network security rules and host-level firewalls
- A systematic method for finding the actual point of failure when a pipeline that looks correctly configured still isn't delivering data
- A cleanly built, least-privilege environment: management and forwarding ports scoped to only the sources that need them
- Turning ingested log data into operational visibility (a dashboard) and automated detection (a scheduled alert), not just confirming ingestion works

---

## Lab Objectives

- Configure a Splunk Universal Forwarder's `inputs.conf` to monitor the Security, System, and Application Windows Event Logs and route them to a dedicated index
- Write a PowerShell script to generate realistic test events: failed logons, a successful logon, service restarts, custom application log entries, and an account lockout
- Validate Windows Advanced Audit Policy settings required for logon and account lockout events to actually be written
- Diagnose a real "zero events indexed" incident end to end: local event generation, network path (Azure NSG, Linux host firewall), Splunk receiving configuration, and Splunk's own config-file loading behavior
- Identify and fix the actual root cause using Splunk's `btool` CLI rather than assuming the problem based on symptoms alone
- Deploy a Splunk indexer on Ubuntu in Azure with network security rules scoped to least privilege (admin-only Web/SSH, VNet-only forwarding)
- Install and configure a Universal Forwarder end to end, from receiving configuration through index creation to validated test data
- Build a security-monitoring dashboard and a scheduled alert on top of validated log data

---

## Tools Used

- Splunk Universal Forwarder (Windows)
- Splunk Enterprise (Ubuntu indexer, Azure)
- PowerShell (`Get-WinEvent`, `auditpol`, `net use`, `Test-NetConnection`)
- Windows Advanced Audit Policy (`auditpol`)
- Azure Network Security Groups (NSG)
- Linux host firewall (firewalld / ufw / iptables)
- Splunk `btool` CLI
- Splunk dashboards (Classic) and scheduled alerting
- PuTTY (SSH), `dpkg` (Splunk package install)

---

## Skills Demonstrated

- `inputs.conf` configuration for `WinEventLog://Security`, `WinEventLog://System`, and `WinEventLog://Application`, including index routing
- Windows Advanced Audit Policy configuration and validation for Logon/Logoff, Account Lockout, and Credential Validation subcategories
- Generating verifiable Windows Security events (4624, 4625, 4740, 7036, custom 1001) using NTLM authentication against the local SAM (`net use`) rather than relying on interactive-session-dependent methods
- Layered troubleshooting methodology: isolating "is the event being generated" from "is the network path open" from "is Splunk actually loading this configuration"
- Reading Splunk forwarder connection logs (`splunkd.log`) to confirm TCP-level connectivity to an indexer
- Using `btool inputs list --debug` to see the true, merged, effective configuration Splunk is running, not just what a single file on disk says
- Root-causing a silent configuration failure (a config file that Splunk never loaded because its filename didn't exactly match what it expects)
- Scoping Azure NSG rules by least privilege: admin-IP-only for management ports (8000, 22), VNet-only for the data-forwarding port (9997)
- Installing Splunk Enterprise on Ubuntu via `dpkg`, enabling boot-start, and configuring receiving and index creation through Splunk Web
- Building Classic dashboards (account activity, process activity, login trends, after-hours logins) and a scheduled cron-based alert (`*/15 * * * *`) with a results-based trigger condition

---

## Lab Architecture

```
Azure Resource Group (Splunk lab)
│
├── Windows Server VM
│   ├── Security / System / Application Event Logs
│   │   └── Generated by Windows-SecurityEvent-Generator.ps1
│   │       (failed logons, successful logon, service restarts,
│   │        application log entries, account lockout)
│   │
│   └── Splunk Universal Forwarder
│       └── inputs.conf (scripts/universal-forwarder-inputs.conf)
│           [WinEventLog://Security]     index = windows_logs
│           [WinEventLog://System]       index = windows_logs
│           [WinEventLog://Application]  index = windows_logs
│
├── Network Security Group
│   ├── 8000 (Splunk Web) — admin IP only
│   ├── 22   (SSH)        — admin IP only
│   └── 9997 (Forwarding) — VNet 10.0.0.0/24 only, never public
│
└── Splunk Indexer (Ubuntu VM)
    ├── Receiving configured and listening on 9997
    ├── index = windows_logs  ← search target
    ├── Dashboard: Windows Security Overview
    │   (account activity, top processes, login activity, after-hours logins)
    └── Alert: High-Privileged Logon Count
        (scheduled every 15 min, triggers when results > 0)
```

---

## Lab Status

- [x] Universal Forwarder configured to monitor Security, System, and Application logs
- [x] Test event generation script written and validated
- [x] Windows audit policy confirmed correctly configured
- [x] Real Windows Security events confirmed generated (`Get-WinEvent` record count validation)
- [x] Network path validated (Azure NSG, Linux firewall, Splunk receiving config, forwarder connection logs)
- [x] Root cause of a real "zero events indexed" incident identified and resolved
- [x] Data confirmed flowing into `index=windows_logs` after the fix
- [x] Splunk indexer rebuilt on Ubuntu with least-privilege NSG rules
- [x] Universal Forwarder installed and validated against the rebuilt indexer
- [x] Windows Security Overview dashboard built and populated with live data
- [x] High-Privileged Logon Count alert created, scheduled, and validated

---

## Completed Labs

| Lab | Description | Status |
| --- | --- | --- |
| [Lab 01](labs/01-windows-event-log-splunk-pipeline.md) | Build and Troubleshoot a Windows Event Log to Splunk Pipeline | Complete |
| [Lab 02](labs/02-splunk-deployment-dashboards-and-alerts.md) | Deploy Splunk, Configure Windows Log Forwarding, and Build Dashboards and Alerts | Complete |

---

## Security Note

Lab 01 has no screenshot evidence; that documentation is based on PowerShell console output, Windows Event Log record counts, and Splunk `splunkd.log` / `btool` output. Lab 02 includes sanitized screenshots (`screenshots/lab-02-splunk-deployment-dashboards-and-alerts/`) with live public IP addresses, an Azure subscription ID, and a tenant domain redacted from browser address bars and one SSH login line; internal/private VNet IP addresses (e.g. `10.0.0.5`) were left visible since they have no external exposure. Management access (Splunk Web, SSH) and the data-forwarding port were scoped to the admin's own IP and the internal VNet range respectively — never exposed to the public internet. No real user credentials were used; the lockout and failed-logon testing used a disposable local test account (`labtest.user`) created and removed by the script itself.

---

## Key Takeaway

This project demonstrates the full lifecycle of a SIEM log pipeline, not just one slice of it: building the environment correctly and securely from the start (Lab 02), and methodically troubleshooting it end to end when it silently broke (Lab 01) — ruling out host audit policy, event generation, cloud network rules, host firewall, and Splunk's own receiving config before finding the actual root cause, a single mistyped filename. Turning validated log data into a dashboard and a scheduled alert closes the loop from "logs are arriving" to "someone gets notified when it matters," which is the actual point of running a SIEM.

---

## Video Walkthrough

- [Deploy and Configure Splunk for Windows Event Log Monitoring in Azure](https://loom.com/share/4e2bb427a6794b88b079367dcf052ab7) — Lab 02
