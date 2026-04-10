# Phase 2 — Wazuh SIEM Deployment

Deploying the open-source Wazuh SIEM as the heart of the lab, then connecting
the Windows victim as the first monitored endpoint.

## Deployment Method
Used the official **Wazuh 4.14.4 OVA** for speed — pre-built on Amazon
Linux 2023 with Wazuh manager, indexer (OpenSearch), and dashboard
(OpenSearch Dashboards) already configured.

### Resource tuning
Default OVA ships with 4 CPU / 8 GB RAM — too heavy for a 16 GB host. Downsized
at import time to **2 CPU / 4 GB RAM** which still runs the full stack (tight
but workable; may bump to 5 GB if indexer feels sluggish).

## Network Configuration

| Component | Value |
|---|---|
| Wazuh VM IP | `10.0.10.7` (DHCP on SOCLab) |
| Agent → Manager | TCP/UDP 1514 (event ingestion) |
| Agent enrollment | TCP 1515 |
| Wazuh API | TCP 55000 |
| Dashboard (HTTPS) | TCP 443 inside VM |

### Host access via port forwarding
Since the SOCLab network is isolated, the host laptop cannot directly reach
`10.0.10.7`. Configured a VirtualBox NAT Network port forwarding rule:

| Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
|---|---|---|---|---|---|
| Wazuh-HTTPS | TCP | 127.0.0.1 | 8443 | 10.0.10.7 | 443 |

Dashboard then accessible at **https://127.0.0.1:8443** (self-signed cert
warning is expected — normal for internal tools).

## Windows Agent Deployment
Used the Wazuh dashboard's "Deploy new agent" wizard to generate the install
command:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi `
  -OutFile ${env:tmp}\wazuh-agent.msi; `
msiexec.exe /i ${env:tmp}\wazuh-agent.msi /q `
  WAZUH_MANAGER='10.0.10.7' WAZUH_AGENT_NAME='windows-victim'
NET START WazuhSvc
```

Ran in elevated PowerShell on the Windows VM. Agent appeared in the dashboard
as **Active** within ~60 seconds.

## First Alerts Generated
To validate the pipeline, failed a Windows login 4 times deliberately, then
logged in successfully twice. Wazuh picked it up cleanly — see the first
threat hunting report in [`../reports/phase2-first-threat-hunting-report.pdf`](../reports/phase2-first-threat-hunting-report.pdf).

### Key rules that fired

| Rule ID | Description | Why it matters |
|---|---|---|
| 60122 | Logon Failure — Unknown user or bad password | My 4 deliberate failures ✅ |
| 60118 | Windows Workstation Logon Success | Recovery logins |
| 67028 | Special privileges assigned to new logon | Fired when I opened elevated PS |
| 67023 | Non-service account logged off | Normal logoff events |
| 501 | New Wazuh agent connected | The moment the agent came online |
| 503 | Wazuh agent started | Agent service startup |

### MITRE ATT&CK mapping (automatic)
Wazuh auto-mapped the alerts to:
- **T1078** Valid Accounts
- **T1531** Account Access Removal
- **T1562** Disable or Modify Tools
- **T1484** Domain Policy Modification

## CIS Benchmark Findings
Wazuh's SCA module auto-ran CIS benchmarks on the Windows endpoint and the
Wazuh host itself on first scan:

| System | Benchmark | Score |
|---|---|---|
| Windows 11 | CIS Microsoft Windows 11 Enterprise v3.0.0 | **26%** |
| Amazon Linux 2023 (Wazuh host) | CIS Amazon Linux 2023 v1.0.0 | **43%** |

Both scores are a remediation backlog, not a pass. Major Windows gaps
identified: no account lockout policy, guest account enabled, Basic auth
allowed, RDP enabled, Cortana + Clipboard Sync on, no audit policy
hardening. These will be fixed as part of an upcoming hardening exercise
(before/after compliance comparison = great interview story).

## Status
✅ **SIEM is operational.** Telemetry flowing from Windows endpoint, alerts
being enriched with MITRE mapping, CIS compliance scanning running in the
background, reports exportable as PDF.

Snapshot taken: `phase2-complete-agents-connected`

## Next — Phase 3
- Install Ubuntu Wazuh agent (DEB package)
- Deploy Sysmon on Windows with SwiftOnSecurity config for rich process
  telemetry (Event ID 1 command-line, Event ID 3 network connections,
  Event ID 11 file create, etc.)
- Deploy Neo23x0 auditd rules on Ubuntu
- Verify enriched events appear in Wazuh
