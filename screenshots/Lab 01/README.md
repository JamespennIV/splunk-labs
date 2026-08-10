# Screenshot Index — Lab 01: Splunk Pipeline Troubleshooting and Deployment (Part 2)

Part 1 of this lab (the original "zero events indexed" troubleshooting incident) has no screenshot evidence; it's documented directly in the lab writeup via terminal output, Windows Event Log record counts, and Splunk `splunkd.log` / `btool` excerpts. These screenshots cover Part 2: rebuilding the indexer on Ubuntu, redeploying the Universal Forwarder, and building the dashboard and alert.

All screenshots are cropped to remove the Windows taskbar. Four screenshots (09–12) also had a remote-session title bar cropped off the top, since it displayed the forwarder VM's live public IP address. Browser address bars were redacted wherever they exposed a live public IP address, an Azure subscription ID, or a tenant domain name. No other content, including the webcam overlay, was altered.

| # | File | What it shows |
| --- | --- | --- |
| 01 | `01-delete-old-vm.jpg` | Deleting the VM used in Part 1 (force delete, all associated resources) to rebuild cleanly in a new resource group. |
| 02 | `02-networking-initial-basic-nsg.jpg` | First pass at VM networking: Basic NSG with SSH (22) allowed from all IP addresses (Azure's own warning banner visible) and a new, non-default subnet — corrected in the next step. |
| 03 | `03-networking-advanced-nsg-corrected.jpg` | Corrected networking: Advanced NSG (`VM2_Splunk`), reverted to the existing default `10.0.0.0/24` subnet. |
| 04 | `04-review-create-vm.jpg` | Review + create step for the new Ubuntu VM. |
| 05 | `05-ssh-login-banner.jpg` | SSH login banner on first connection to the new Ubuntu 24.04 VM (internal IP `10.0.0.5` visible — private VNet address, not exposed). |
| 06 | `06-splunk-download-install.jpg` | Downloading the Splunk `.deb` package and installing it — includes two real failed `dpkg -i` attempts (wrong filename/version) before the correct package name was used. |
| 07 | `07-splunk-service-starting.jpg` | Splunk starting for the first time: port checks, directory creation, cert generation, confirming the web interface is available. |
| 08 | `08-splunk-web-login.jpg` | Splunk Web login page. |
| 09 | `09-forwarder-vm-edge-newtab.jpg` | Browser on the Windows forwarder VM, mid-session (context screenshot). |
| 10 | `10-forwarder-inputconf-downloaded.jpg` | The Universal Forwarder's `local` config folder, with a newly downloaded config file about to be opened for editing. |
| 11 | `11-forwarder-inputconf-edited.jpg` | The `WinEventLog://Security`, `System`, and `Application` stanzas written out in the editor, each routed to the `windows_logs` index. |
| 12 | `12-forwarder-local-config-folder.jpg` | The forwarder's `system\local` folder with the finished config file saved. **Note:** the file is named `input.conf` here — the exact same one-character naming mistake made (and fixed) earlier in this lab's Part 1. Since later screenshots (13, 14) confirm data was flowing into Splunk, this was corrected to `inputs.conf` before the forwarder was restarted, though that specific rename wasn't captured in a screenshot. |
| 13 | `13-search-after-hours-validation.jpg` | Validating the pipeline: an after-hours logon search against `index=windows_logs`, returning real matched events. |
| 14 | `14-windows-security-overview-dashboard.jpg` | The finished "Windows Security Overview" dashboard, showing the Account Activity and Top Processes panels populated with live data. |

## Sanitization applied

- Cropped: Windows taskbar (all images).
- Cropped: remote-session title bar showing the forwarder VM's live public IP (images 09–12).
- Redacted: browser address bar wherever it exposed a live public IP address, Azure subscription ID, or tenant domain (images 01, 08, 13, and the Delete VM2 subscription ID field on image 01).
- Redacted: the SSH login line containing the indexer VM's live public IP (image 05).
- Not altered: internal/private VNet IP addresses (e.g. `10.0.0.5`), hostnames, on-screen application content, or the webcam overlay.
