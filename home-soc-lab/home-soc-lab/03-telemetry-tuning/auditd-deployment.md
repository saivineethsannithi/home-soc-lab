# Phase 3 Closeout — auditd on Ubuntu (Linux Telemetry)

> Companion to the Sysmon Windows deployment. This is the step that brings
> Ubuntu telemetry up to the same level as the Windows victim and officially
> closes Phase 3.

## Goal

Get rich Linux telemetry — process execution, file access, privilege
escalation, sensitive file modifications — flowing into Wazuh, mapped to
a community-maintained ruleset (Florian Roth's Neo23x0 audit rules).

## Stack

| Component | Purpose |
|---|---|
| **auditd** | Linux kernel-level system call auditor (built into Ubuntu, just needs rules) |
| **Neo23x0 audit ruleset** | Community gold-standard auditd rules, MITRE ATT&CK aware |
| **Wazuh agent** | Reads `/var/log/audit/audit.log` and forwards events to the manager |

---

## Step-by-step walkthrough

### 1. Install auditd on Ubuntu

Ubuntu Server 24.04 doesn't ship with auditd by default — had to install
it first:

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
```

Output pulled in `libauparse0t64`, `auditd`, and `audispd-plugins`,
created the systemd service unit, and finished cleanly with `Running
kernel seems to be up-to-date.`

### 2. Enable and start the auditd service

```bash
sudo systemctl enable --now auditd
```

📸 **See `screenshots/auditd-systemctl-enable.png`** — output shows:
```
Synchronizing state of auditd.service with SysV service script
with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable auditd
```

The service is now persistent across reboots and running immediately.

### 3. Download the Neo23x0 community audit ruleset

```bash
cd /tmp
wget https://raw.githubusercontent.com/Neo23x0/auditd/master/audit.rules
```

📸 **See `screenshots/auditd-neo23x0-wget.png`** — wget output:
```
Resolving raw.githubusercontent.com... 185.199.111.133
Connecting to raw.githubusercontent.com|185.199.111.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 31963 (31K) [text/plain]
Saving to: 'audit.rules'
audit.rules    100%[=====================================>]   31.21K
2026-04-11 08:31:50 (4.08 MB/s) - 'audit.rules' saved [31963/31963]
```

31,963 bytes of community-tested audit rules covering Linux-relevant
MITRE ATT&CK techniques. Same author as the Sigma rules used in Phase 5.

### 4. Install and load the ruleset

```bash
sudo cp /tmp/audit.rules /etc/audit/rules.d/audit.rules
sudo augenrules --load
```

**Important note about the load output:** `augenrules --load` produced
~50 noisy "There was an error in line X" / "No such file or directory"
messages. **These are expected and harmless** — the Neo23x0 ruleset is
a superset designed to load on every Linux distro, so distro-specific
rules (paths to binaries that don't exist on Ubuntu, kernel modules
not present, etc.) are silently skipped.

### 5. Verify rules actually loaded

```bash
sudo auditctl -l | wc -l
```

**Result: 354 rules loaded.** That's deeper visibility than most
production Linux servers run with — typical real-world deployments use
50–100 audit rules. With 354, this Ubuntu VM has very wide coverage of
syscalls, file accesses, privilege changes, and execution events.

### 6. Trigger test events

📸 **See `screenshots/auditd-test-events.png`** — discovery activity
simulation (the same kind of commands an attacker runs in the first 60
seconds of a compromise):

```bash
id
# uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),4(adm),
#   24(cdrom),27(sudo),30(dip),46(plugdev),101(lxd)

uname -a
# Linux ubuntu-victim 6.8.0-107-generic #107-Ubuntu SMP
#   PREEMPT_DYNAMIC Fri Mar 13 19:51:50 UTC 2026 x86_64 GNU/Linux

ps aux | head
# (running processes - init, kthreadd, kworker, etc.)

netstat -tulpn 2>/dev/null || ss -tulpn
# Listening sockets: 127.0.0.53:53 (DNS), 10.0.10.5:68 (DHCP),
#   0.0.0.0:22 (SSH)

sudo touch /etc/passwd
# (sensitive file modification - audited)
```

Each command represents a MITRE ATT&CK Discovery technique (TA0007) that
auditd is now configured to capture.

### 7. Tell the Wazuh agent to read the audit log

Same pattern as the Sysmon `<localfile>` block on Windows — the agent
has to be told where to look:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Added this block alongside the existing localfile entries:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

Saved with **Ctrl+O → Enter → Ctrl+X**.

### 8. Restart the Wazuh agent

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

Confirmed `active (running)`.

### 9. Verify in the Wazuh dashboard

📸 **See — the Threat Hunting
view filtered to `agent.id: 002` (ubuntu-victim):

<img width="1918" height="941" alt="Screenshot 2026-04-11 142029" src="https://github.com/user-attachments/assets/7c592604-7bbb-47a1-a810-cadfef4f2a33" />


| Metric | Value |
|---|---|
| Total events | **421** |
| Level 12+ alerts | 0 (clean baseline, nothing malicious) |
| Authentication failures | 6 |
| Authentication successes | 22 |

Top 10 alert groups firing on the Ubuntu side:
`sca`, `syslog`, `pam`, `authentication_success`, `sudo`, `dpkg`,
`apparmor`, `config_changed`, `local`, `authentication_failed`.

This is multi-source telemetry — the Linux side now matches the Windows
side in breadth.

---


[wazuh-module-agents-002-general-1775897392.pdf](https://github.com/user-attachments/files/26644832/wazuh-module-agents-002-general-1775897392.pdf)
## What Wazuh detected (Ubuntu Phase 3 closeout report)




### Top alerts

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 5502 | PAM: Login session closed | 3 | 23 |
| 5501 | PAM: Login session opened | 3 | 22 |
| 5402 | Successful sudo to ROOT executed | 3 | 21 |
| **52004** | **AppArmor DENIED mknod operation** | **4** | **10** |
| 2902 | New dpkg (Debian Package) installed | 7 | 5 |
| 2904 | Dpkg (Debian Package) half configured | 7 | 5 |

**Why rule 52004 is interesting:** AppArmor is Ubuntu's mandatory access
control system. When it fires DENIED events, something tried to create a
device file or socket and was blocked by policy. This is exactly the kind
of low-level activity that catches privilege escalation attempts. **Caught
10 times during routine lab activity** — proof that mandatory access
control is silently doing its job in the background.

**Why the dpkg rules fired:** I installed `auditd` and `audispd-plugins`,
which Wazuh tracked as package state changes. In a real environment,
unexpected package installs are a strong indicator of compromise (attackers
often install tools post-exploitation).

### The most interesting finding — dynamic compliance state changes

The Wazuh CIS Benchmark scanner re-ran after auditd came online and
detected changes in real time:

**Rule group 19014 — "Status changed from 'not applicable' to failed"
(Level 9, 16 hits):**
CIS audit-rule checks that *used to be skipped* but became applicable
because auditd is now running. They fail because the Neo23x0 ruleset
doesn't perfectly map to every CIS-required audit rule. **The SIEM's
detection coverage literally got wider mid-day, and it noticed.**

**Rule group 19010 — "Status changed from failed to passed"
(Level 3, 11 hits):**
CIS hardening checks that **transitioned to PASS** because installing
auditd improved the security posture:
- `Ensure auditd packages are installed` ✅
- `Ensure auditd service is enabled and active` ✅
- `Ensure sshd HostbasedAuthentication is disabled` ✅
- `Ensure sshd PermitEmptyPasswords is disabled` ✅
- `Ensure sshd UsePAM is enabled` ✅
- `Ensure sshd IgnoreRhosts is enabled` ✅
- `Ensure sshd LogLevel is configured` ✅
- `Ensure sshd MaxSessions is configured` ✅
- `Ensure sshd PermitUserEnvironment is disabled` ✅
- `Ensure sshd GSSAPIAuthentication is disabled` ✅
- `Ensure audit tools owner is configured` ✅

**The system became more secure as a side effect of telemetry deployment,
and Wazuh caught the improvement automatically.** That's mature SIEM
behavior — exactly what real production SIEMs do.

### Group totals (page 9 of report)

```
sca                     320  ← CIS Benchmark scanner
syslog                   96  ← System logs
pam                      46  ← Authentication framework
authentication_success   22  ← Successful logins
sudo                     22  ← Every sudo command
dpkg                     13  ← Package install/remove
apparmor                 10  ← MAC denials
config_changed           10  ← Filesystem changes
local                    10  ← Local rules
authentication_failed     6  ← Brute force indicators
invalid_login             4  ← Bad usernames
sshd                      4  ← SSH service events
ossec                     3  ← Wazuh self-monitoring
audit                     2  ← auditd events ✅
audit_selinux             2  ← MAC system events
access_control            1  ← Permission changes
```

**16 distinct rule groups firing** — production-grade Linux telemetry
breadth.

---

## Phase 3 — final scoreboard

| Component | Status |
|---|---|
| Ubuntu Wazuh agent | ✅ active, 421 events flowing |
| Sysmon on Windows | ✅ active, 928+ events |
| Wazuh Sysmon ingestion | ✅ sysmon_eid1, eid11, process-anomalies firing |
| Sysmon Level 12+ detections | ✅ T1036.005 + T1546.011 captured |
| Sysmon triage writeup | ✅ `investigation-high-severity-sysmon.md` |
| auditd ruleset deployed | ✅ 354 rules loaded |
| Wazuh auditd ingestion | ✅ audit + audit_selinux groups firing |
| Linux compliance scanning | ✅ CIS Ubuntu 24.04 LTS active |
| Phase 3 closeout report | ✅ `phase3-complete-ubuntu-auditd-active.pdf` |

**Phase 3 is COMPLETE.** ✅

---

## Screenshots in this folder

| File | What it shows |
|---|---|
| `auditd-systemctl-enable.png` | `systemctl enable --now auditd` synchronizing the service unit with SysV and enabling autostart |
| `auditd-neo23x0-wget.png` | `wget` pulling the Neo23x0 ruleset from raw.githubusercontent.com — 31,963 bytes downloaded successfully |
| `auditd-test-events.png` | Discovery activity simulation (`id`, `uname -a`, `ps aux`, `ss -tulpn`, `sudo touch /etc/passwd`) on Ubuntu 24.04, kernel 6.8.0-107-generic |
| `wazuh-ubuntu-421-events.png` | Wazuh dashboard filtered to `agent.id: 002` showing 421 events, 6 auth failures, 22 auth successes, 10 alert groups |

---

## Lessons learned

1. **Default Ubuntu Server doesn't include auditd.** Had to `apt install
   auditd audispd-plugins` first. Worth knowing — production Linux hosts
   often have it pre-installed via security baselines, but minimal
   installs do not.

2. **The Neo23x0 ruleset's load errors look terrifying but are normal.**
   The ruleset is a superset designed for every Linux distro, and
   distro-specific rules silently skip. Verify with `auditctl -l | wc -l`
   instead of trusting the noisy load output. Result: 354 rules loaded
   despite ~50 skip messages.

3. **Wazuh dynamically re-runs compliance scans.** When system state
   changes (like installing auditd), CIS checks transition between
   pass/fail/applicable states automatically. The 19010 / 19014 rule
   groups capture these transitions — making the SIEM aware of security
   posture drift in real time.

4. **The same `<localfile>` pattern works on both Windows and Linux.**
   Both Sysmon and auditd needed an explicit localfile block in
   `ossec.conf` — different syntax (Windows `eventchannel` vs Linux
   audit log path), same underlying concept. Ingestion is opt-in.

5. **Sometimes installing a security tool improves your CIS score
   automatically.** I didn't run any hardening commands — just installed
   auditd — and 11 SSH hardening checks transitioned from FAIL to PASS
   in the next scan cycle. Good telemetry tools and good baselines
   reinforce each other.

---

## Snapshots taken

| VM | Snapshot |
|---|---|
| ubuntu-victim | `phase3-complete-auditd-active` |
| windows-victim | `phase3-sysmon-deployed` |
| wazuh-server | `phase3-rich-telemetry` |

Each snapshot is the rollback point for Phase 4. If anything breaks
during attack simulations, the lab can return to clean Phase 3 state
in 10 seconds.

---

## What this enables for Phase 4

Now that both endpoints are fully instrumented, Phase 4 (attack
simulations) will run against a SIEM that can actually catch what gets
thrown at it. Specifically:

- **Nmap recon (T1046)** → caught by sshd + AppArmor on Linux, by Sysmon
  network connect on Windows
- **SSH brute force (T1110)** → caught by `authentication_failed` and
  `invalid_login` rule groups on Ubuntu
- **PowerShell encoded execution (T1059.001)** → caught by Sysmon EID 1
  with full command line on Windows
- **New local user creation (T1136)** → caught by sudo + audit groups on
  Linux, Sysmon + Windows audit on Windows
- **SUID binary creation (T1548)** → caught by auditd file modification
  rules on Ubuntu
- **Persistence via systemd or scheduled task** → caught by config_changed
  + Sysmon EID 13 (registry) on Windows, audit on Linux

Every Phase 4 attack should land in this telemetry pipeline.

---

## Sources

- **auditd** — built into Ubuntu (`apt install auditd`)
- **Neo23x0 audit ruleset** — github.com/Neo23x0/auditd
- **Wazuh agent docs (Linux audit integration)** — documentation.wazuh.com
- **CIS Ubuntu 24.04 LTS Benchmark** — auto-deployed via Wazuh SCA module

---

*Phase 3 closeout: April 11, 2026. Both Windows and Linux endpoints now
streaming rich telemetry into Wazuh. Ready for Phase 4 attack simulations.*
