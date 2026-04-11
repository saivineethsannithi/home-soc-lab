# Phase 3 Closeout — auditd on Ubuntu

> Linux-side companion to the Sysmon Windows deployment. This is the step
> that brings Ubuntu telemetry up to the same level as the Windows victim
> and officially closes Phase 3.

## Goal

Get rich Linux telemetry — process execution, file access, privilege
escalation, sensitive file modifications — flowing into Wazuh, mapped
to a community-maintained ruleset.

## Toolchain

| Component | Purpose |
|---|---|
| **auditd** | Linux kernel-level system call auditor (built into Ubuntu, just needs rules) |
| **Neo23x0 audit ruleset** | Florian Roth's gold-standard auditd ruleset, MITRE ATT&CK tagged |
| **Wazuh agent** | Reads `/var/log/audit/audit.log` and forwards events to the manager |

---

## Step-by-step

### 1. Install auditd (it wasn't already present on Ubuntu Server 24.04)

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
```

The package install pulled in `libauparse0t64`, `auditd`, and
`audispd-plugins`, then created the systemd service unit and processed
triggers cleanly. **See `screenshots/auditd-package-install.png`** for
the full apt output — clean install with no errors.

### 2. Enable and start auditd

```bash
sudo systemctl enable --now auditd
```

Output: `Synchronizing state of auditd.service with SysV service script ...`
**See `screenshots/auditd-systemctl-enable.png`** — the service was
created and enabled in one step.

### 3. Download the Neo23x0 community ruleset

```bash
cd /tmp
wget https://raw.githubusercontent.com/Neo23x0/auditd/master/audit.rules
```

**See `screenshots/auditd-neo23x0-wget.png`** — file downloaded
successfully, 31,963 bytes from raw.githubusercontent.com. This is
~400 lines of community-tested audit rules covering Linux-relevant
MITRE ATT&CK techniques.

### 4. Install and load the ruleset

```bash
sudo cp /tmp/audit.rules /etc/audit/rules.d/audit.rules
sudo augenrules --load
```

**Important note about the load output:**
The `augenrules --load` command produced ~50 "There was an error in
line X" / "No such file or directory" messages. These are **expected
and harmless** — the Neo23x0 ruleset is a superset designed to load
on every Linux distro, so distro-specific rules (paths to binaries
that don't exist on Ubuntu, kernel modules not present, etc.) are
silently skipped.

### 5. Verify rules actually loaded

```bash
sudo auditctl -l | wc -l
```

**Result: 354 rules loaded.** That's more than I see on most
production Linux servers — typical real-world deployments run with
50–100 audit rules. With 354, this Ubuntu VM has very deep visibility
into syscalls, file accesses, privilege changes, and execution events.

### 6. Test that auditd is capturing events

```bash
sudo cat /etc/shadow > /dev/null
id
uname -a
ps aux | head
netstat -tulpn 2>/dev/null || ss -tulpn
sudo touch /etc/passwd
```

**See `screenshots/auditd-test-events.png`** — full output from a
discovery activity simulation:
- `id` showed UID 1000, groups including `sudo`, `lxd`, `cdrom`
- `uname -a` showed kernel `6.8.0-107-generic` on Ubuntu 24.04
- `ps aux` showed running processes
- `ss -tulpn` showed listening sockets including `0.0.0.0:22` (SSH)
- `sudo touch /etc/passwd` triggered a sensitive file modification event

Each of those commands is exactly the kind of thing an attacker would
run during the Discovery phase of an attack chain (MITRE TA0007).

### 7. Tell the Wazuh agent to read auditd's log

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Added this `<localfile>` block alongside the existing ones (same
pattern as the Sysmon `<localfile>` block on Windows):

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

Saved with Ctrl+O / Enter / Ctrl+X.

### 8. Restart the Wazuh agent

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

Confirmed `active (running)`.

### 9. Verify in the Wazuh dashboard

**See `screenshots/wazuh-ubuntu-421-events.png`** — the Threat Hunting
view filtered to `agent.id: 002` (ubuntu-victim):

- **Total events:** 421
- **Level 12 or above alerts:** 0 (clean baseline, nothing malicious)
- **Authentication failures:** 6
- **Authentication successes:** 22

Rule groups firing on the Ubuntu side:
`sca`, `syslog`, `pam`, `authentication_success`, `sudo`, `dpkg`,
`apparmor`, `config_changed`, `local`, `authentication_failed`.

This is multi-source telemetry — the Linux side now matches the
Windows side in breadth.

---

## What Wazuh detected (final Phase 3 Ubuntu report)

**Source:** `../reports/phase3-complete-ubuntu-auditd-active.pdf`

### Top alerts

| Rule ID | Description | Level | Count |
|---|---|---|---|
| 5502 | PAM: Login session closed | 3 | 23 |
| 5501 | PAM: Login session opened | 3 | 22 |
| 5402 | Successful sudo to ROOT executed | 3 | 21 |
| 52004 | AppArmor DENIED mknod operation | **4** | **10** |
| 2902 | New dpkg (Debian Package) installed | 7 | 5 |
| 2904 | Dpkg (Debian Package) half configured | 7 | 5 |

**Why rule 52004 is interesting:** AppArmor is Ubuntu's mandatory access
control. When it fires DENIED events, something tried to create a device
file or socket and was blocked by policy. This is exactly the kind of
low-level activity that catches privilege escalation attempts. **Caught
10 times during routine lab activity** — proof that mandatory access
control is doing its job.

**Why the dpkg rules fired:** I installed `auditd` and `audispd-plugins`,
which Wazuh tracked as package state changes. In a real environment,
unexpected package installs are a strong indicator of compromise.

### Compliance scanning dynamic status changes

The most interesting observation in this report is the **status
transitions** that fired automatically. Wazuh re-ran the CIS Benchmark
after auditd came online, and detected changes:

**Rule group 19014 — "Status changed from 'not applicable' to failed"
(Level 9, ~16 hits):**
These are CIS audit-rule checks that *used to be skipped* but are now
applicable because auditd is running. They fail because the Neo23x0
ruleset doesn't perfectly map to every CIS-required audit rule. This
means **Wazuh's detection coverage literally got better mid-day, and the
SIEM noticed.**

**Rule group 19010 — "Status changed from failed to passed"
(Level 3, 11 hits):**
These are CIS hardening checks that **transitioned to PASS** because
installing auditd improved the security posture. Examples:
- `Ensure auditd packages are installed` ✅
- `Ensure auditd service is enabled and active` ✅
- `Ensure sshd HostbasedAuthentication is disabled` ✅
- `Ensure sshd PermitEmptyPasswords is disabled` ✅
- `Ensure sshd UsePAM is enabled` ✅
- (and 6 more SSH hardening checks)

**The system actually got more secure as a side effect of telemetry
deployment, and Wazuh caught the improvement.** That's mature SIEM
behavior, and a great talking point for interviews.

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

**16 distinct rule groups firing** — that's the kind of breadth a real
production Linux SOC operates with.

---

## Phase 3 — final scoreboard

| Component | Status |
|---|---|
| Ubuntu Wazuh agent | ✅ active, 421 events flowing |
| Sysmon on Windows | ✅ active, 928 events earlier today |
| Wazuh Sysmon ingestion | ✅ sysmon_eid1, eid11, process-anomalies firing |
| Sysmon Level 12+ detections | ✅ T1036.005 + T1546.011 captured |
| Sysmon triage writeup | ✅ investigation-high-severity-sysmon.md |
| auditd ruleset deployed | ✅ 354 rules loaded |
| Wazuh auditd ingestion | ✅ audit + audit_selinux groups firing |
| Linux compliance scanning | ✅ CIS Ubuntu 24.04 LTS active |
| Phase 3 closeout report | ✅ phase3-complete-ubuntu-auditd-active.pdf |

**Phase 3 is COMPLETE.**

---

## Lessons learned

1. **Default Ubuntu Server doesn't ship with auditd installed.** Had to
   `apt install auditd audispd-plugins` first. Worth knowing — production
   Linux hosts often have it pre-installed via security baselines, but
   minimal installs do not.

2. **The Neo23x0 ruleset's load errors look terrifying but are normal.**
   The ruleset is a superset designed for every Linux distro, and
   distro-specific rules silently skip. Verify with `auditctl -l | wc -l`
   instead of trusting the load output.

3. **Wazuh dynamically re-runs compliance scans.** When system state
   changes (like installing auditd), CIS checks transition between
   pass/fail/applicable states automatically. The 19010 / 19014 rule
   groups capture these transitions — making the SIEM aware of security
   posture drift in real time.

4. **The same `<localfile>` pattern works on both Windows and Linux.**
   Both Sysmon and auditd needed an explicit localfile block in
   ossec.conf — different syntax (Windows eventchannel vs Linux audit
   log path), same underlying concept. Ingestion is opt-in.

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
