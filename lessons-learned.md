# Lessons Learned — Splunk SIEM Log Pipeline Lab

## What went well

- Building the test event generation script around real Windows authentication mechanisms (`net use` against the local SAM) instead of session-dependent methods made the events reliably reproducible regardless of how the VM was being accessed.
- Verifying Windows audit policy before assuming anything was wrong with Splunk saved time, since a misconfigured audit policy would have made every downstream check meaningless.
- Independently confirming event generation (checking the Security log's record count before and after a manual test) before touching the network or Splunk config ruled out an entire category of possible causes early.
- Working through the pipeline in a defined order, source, audit policy, network, receiving config, then Splunk's own config loading, made it possible to say with confidence which layers were and weren't the problem at each step.
- Scoping the network security group before installing or configuring Splunk at all — admin-IP-only for Web and SSH, VNet-only for the forwarding port — meant the environment was never briefly exposed while being set up.
- Confirming data was actually flowing into the index before building the dashboard avoided wasted effort building panels against an empty data source.

## What was challenging

- Every individual layer checked out fine on its own, which made it tempting to assume the problem had to be in a layer not yet checked, rather than in something invisible to normal configuration review.
- A disabled `splunklogger` index looked like a plausible routing mismatch early on and cost some time to rule out, before confirming it was an unrelated leftover from earlier configuration.
- A failed `Test-NetConnection` to the forwarder's local management port (8089) initially looked like it might indicate a broader forwarder health problem, when the actual data-forwarding path (port 9997) was working the entire time.
- The actual root cause, a config file named `input.conf` instead of `inputs.conf`, produced no error, warning, or log entry anywhere. Nothing about Splunk's behavior pointed at it directly; it had to be found by comparing the effective configuration (`btool`) against the file that was actually being edited.
- Getting the Splunk `dpkg` install right on the first pass required matching the exact package filename and version; a mismatch fails immediately, which is at least an obvious failure mode compared to the silent `input.conf` issue.
- The `input.conf` vs `inputs.conf` naming mistake from this lab's Part 1 happened again while rebuilding the environment for Part 2, on a completely fresh VM. Having already documented the lesson once didn't stop the same typo from recurring — a reminder that a written lesson only helps if it's actually checked against in the moment, not just read once.

## What I'd do differently next time

- Run `btool inputs list --debug` earlier in the process, right after confirming the target index exists, rather than as a last resort after exhausting every other layer.
- Keep a running checklist of confirmed-good layers during troubleshooting, so it's easier to visually spot that a config-loading check hasn't been done yet.
- Double-check new or renamed config filenames immediately after creating them, since a mismatched filename fails silently with zero indication anything is wrong.
- When a local health check (like the 8089 management port test) fails, explicitly confirm whether that specific port is actually relevant to the problem being diagnosed before spending time on it.

## Next steps for this lab environment

- Add alerting for a Splunk data source that stops reporting entirely, so a future occurrence of this exact failure mode would be caught automatically rather than found by manual search.
- Build a second forwarder instance from scratch to confirm the fixed configuration is reproducible and not specific to this one VM's history.
- Extend the test event generator to cover additional event types (e.g., process creation, PowerShell script block logging) for broader SIEM detection testing.
- Add more scheduled alerts covering other high-value detections (e.g., account lockout spikes, after-hours logon volume) beyond the single High-Privileged Logon Count alert built so far.
