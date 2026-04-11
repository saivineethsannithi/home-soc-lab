# 🛡️ Home SOC Lab

A virtualized Security Operations Center lab built from scratch to practice
**detection engineering, threat hunting, and incident response** using
industry-standard open-source tools. This project simulates a small enterprise
environment where I play both attacker and defender — running real MITRE ATT&CK
techniques and detecting them with a live SIEM.

> **Why this project?** Reading about SOC work is not the same as doing it. This
> lab lets me investigate real alerts from real telemetry on real endpoints,
> producing tangible artifacts (detections, writeups, reports) that demonstrate
> practical SOC analyst skills.

---

## 🏗️ Architecture

```
              ┌─────────────────────────────┐
              │    SOCLab NAT Network       │
              │        10.0.10.0/24         │
              └──────────────┬──────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐   ┌────────▼────────┐   ┌───────▼────────┐
│  Wazuh SIEM   │   │ Windows Victim  │   │ Ubuntu Victim  │
│  (Amazon Lin) │◄──┤   (Win 11)      │   │  (Server 22)   │
│   10.0.10.7   │   │  + Wazuh agent  │   │ + Wazuh agent  │
│               │   │  + Sysmon (next)│   │ + auditd       │
└───────▲───────┘   └─────────────────┘   └────────────────┘
        │
        │                    ┌────────────────┐
        └────────────────────┤ Kali Attacker  │
                             │ 10.0.10.x      │
                             │ (Atomic Red    │
                             │  Team — next)  │
                             └────────────────┘
```
**SOCLab NAT Network**

---
                    10.0.10.0/24
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼─────┐    ┌─────▼─────┐    ┌────▼─────┐
    │ Ubuntu   │    │ Windows   │    │  Kali    │
    │ 10.0.10.4│    │ 10.0.10.5 │    │10.0.10.6 │
    │ (victim) │    │ (victim)  │    │(attacker)│
    └──────────┘    └───────────┘    └──────────┘
                          │
                    ┌─────▼──────┐
                    │   Wazuh    │
                    │ 10.0.10.7  │
                    │   (SIEM)   │
                    └────────────┘
                          ▲
                          │ port forward
                          │ 127.0.0.1:8443 → 443
                    ┌─────┴──────┐
                    │ Your laptop│
                    │   (host)   │
                    └────────────┘

---


**Stack:**
- **Hypervisor:** Oracle VirtualBox 7.x
- **SIEM:** Wazuh 4.14.4 (OVA deployment)
- **Victim (Windows):** Windows 11 Enterprise Evaluation
- **Victim (Linux):** Ubuntu Server 22.04 LTS
- **Attacker:** Kali Linux 2024.x
- **Network:** Isolated NAT Network (10.0.10.0/24)
- **Host access:** Port forwarding 127.0.0.1:8443 → Wazuh dashboard

---

## ✅ Progress

### Phase 1 — Lab Infrastructure (Complete)
- [x] VirtualBox installed, Hyper-V disabled on host, virtualization verified
- [x] Windows 11 victim VM installed + Guest Additions + snapshot
- [x] Ubuntu Server 22.04 victim VM installed + updated + snapshot
- [x] Kali Linux attacker VM imported + networked + snapshot
- [x] SOCLab NAT Network (10.0.10.0/24) created
- [x] All VMs verified with cross-VM ping tests

### Phase 2 — SIEM Deployment (Complete)
- [x] Wazuh 4.14.4 OVA imported with downsized specs (2 CPU / 4 GB RAM)
- [x] Wazuh joined SOCLab network (10.0.10.7)
- [x] Dashboard accessible from host via port forwarding (https://127.0.0.1:8443)
- [x] Windows Wazuh agent installed and reporting (`windows-victim`)
- [x] First threat hunting report generated — **530 events, 4 auth failures detected**
- [x] MITRE ATT&CK mapping verified (Valid Accounts T1078, Disable/Modify Tools T1562)
- [x] CIS Benchmark baseline scan completed (Windows 11 at 26% compliance — remediation backlog)

---

## 📊 Early Results

After deploying Phase 2, the first 24-hour threat hunting report showed:

| Metric | Value |
|---|---|
| Total events processed | **530** |
| Authentication failures detected | 4 |
| Authentication successes logged | 6 |
| MITRE techniques mapped | Valid Accounts, Account Access Removal, Disable/Modify Tools, Domain Policy Modification |
| Windows 11 CIS Benchmark score | 26% (baseline — remediation planned) |
| Amazon Linux CIS score | 43% |

📄 Full report: [`reports/phase2-first-threat-hunting-report.pdf`](reports/phase2-first-threat-hunting-report.pdf)

---

## 🎯 Skills Demonstrated

**SIEM & Detection:**
Wazuh, OpenSearch/Kibana, rule tuning, threat hunting queries, alert triage

**Endpoint Telemetry:**
Windows Event Logs, Sysmon (Phase 3), auditd (Phase 3), log forwarding agents

**Frameworks & Standards:**
MITRE ATT&CK mapping, CIS Benchmarks, Sigma rule format

**Infrastructure:**
VirtualBox, NAT networking, VM snapshots, hypervisor troubleshooting
(Hyper-V/VT-x conflicts, EFI boot issues)

**PowerShell & Bash:**
`New-NetFirewallRule`, `Get-WinEvent`, `Get-NetTCPConnection`, bash pipelines

**Documentation:**
Markdown writeups, screenshots, architecture diagrams, incident reports

---

## 🚧 Lessons Learned (so far)

1. **Debugging the infrastructure IS the learning.** Fighting through Hyper-V
   conflicts, EFI boot loops, and misconfigured NAT taught me how SOC tooling
   actually works under the hood.
2. **Always verify with `ip a` / `ipconfig` — never assume an IP.** Learned the
   hard way during ping tests. This is the same habit that saves SOC analysts
   from chasing ghost findings.
3. **Snapshot before every major change.** Cheap insurance against breaking
   things, which you will.
4. **Principle of least privilege matters in the smallest things** — e.g. the
   `-IcmpType 8` precision on a firewall rule (only allow Echo Request, not all
   ICMP) is exactly the mindset needed for production detection engineering.

---

## 📂 Repository Structure

```
home-soc-lab/
├── README.md                       ← you are here
├── 01-lab-setup/                   ← Phase 1 docs
│   └── lab-setup.md
├── 02-wazuh-deployment/            ← Phase 2 docs
│   └── wazuh-deployment.md
├── 03-telemetry-tuning/            ← Phase 3 (in progress)
├── 04-attack-simulations/          ← Phase 4 (planned)
├── 05-custom-detections/           ← Phase 5 (planned)
│   ├── rules/
│   └── writeups/
├── reports/                        ← generated SIEM reports
│   └── phase2-first-threat-hunting-report.pdf
└── screenshots/                    ← dashboard + alert screenshots
```

---


# Screenshots
Dashboard captures, alert screenshots, architecture diagrams.
Place screenshot files in this folder:


<b>- phase2-wazuh-dashboard-empty.png (before agent)</b>
 <img width="1919" height="942" alt="Screenshot 2026-04-11 024743" src="https://github.com/user-attachments/assets/ece33478-05df-415a-9e66-8e655b8c4cd4" />

<b>- phase2-wazuh-dashboard-agents-active.png (after agent)</b>
<img width="1919" height="944" alt="Screenshot 2026-04-11 025624" src="https://github.com/user-attachments/assets/567e823e-ba25-4546-ad86-0d7dd6d914f2" />

<b>- phase2-mitre-donut-chart.png</b>
<img width="1919" height="954" alt="Screenshot 2026-04-11 133520" src="https://github.com/user-attachments/assets/ef5d8380-b3a2-4a2b-b266-8d2fc1859b2c" />

<b>- phase2-authentication-failures-detected.png</b>
  <img width="1919" height="954" alt="Screenshot 2026-04-11 133446" src="https://github.com/user-attachments/assets/2a13fad9-2a1e-4675-9b29-1704be8e1e0f" />

## 📫 About

Built by **Saivineeth Sannithi (Bunny)** — cybersecurity professional pursuing
SOC analyst roles. MSc Cybersecurity (Teesside University), CompTIA Security+
(SY0-701), Google Cybersecurity Professional certified.

This project is updated regularly as I progress through the phases. Follow
along — every phase produces new artifacts.

**Connect:** Email: - vineethsannithi1998@gmail.com


---

*Lab credentials and IPs are redacted from all public files. This is a
closed, virtualized environment running on an isolated private network
with no real-world exposure.*
