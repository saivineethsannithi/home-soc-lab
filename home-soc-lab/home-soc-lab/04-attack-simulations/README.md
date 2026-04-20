# Phase 4 — Attack Simulations

> Goal: Run real MITRE ATT&CK techniques from Kali against both victim
> endpoints and verify what Wazuh detects — and what it misses.

## Lab Setup

| VM | Role | IP |
|---|---|---|
| kali-attacker | Attacker | 10.0.10.6 |
| windows-victim | Target (Sysmon + Wazuh agent) | 10.0.10.4 |
| ubuntu-victim | Target (auditd + Wazuh agent) | 10.0.10.5 |
| wazuh-server | SIEM | 10.0.10.7 |

---

## Attack 1 — Nmap Port Scan (T1046)

**Target:** Both victims
**Tool:** Nmap from Kali
**Commands:**
```bash
nmap -sV -sC 10.0.10.5
nmap -sV -sC 10.0.10.4
```

**Wazuh result:** No alerts fired.

**Analysis:** Default Wazuh rules don't include port scan detection. Nmap sends TCP SYN packets which are handled at the network layer before reaching application logs that Wazuh monitors. This is a legitimate detection gap — addressed in Phase 5 with a custom correlation rule.

**MITRE:** T1046 — Network Service Scanning
**Verdict:** DETECTION GAP

---

## Attack 2 — SSH Brute Force (T1110.001)

**Target:** Ubuntu victim
**Tool:** Hydra from Kali
**Commands:**
```bash
echo -e "password\n123456\nadmin\nroot\nletmein\npassword123\nLab@1234" > passwords.txt
hydra -l analyst -P passwords.txt 10.0.10.5 ssh -t 4 -V
```

**Wazuh result:** Detected. Multiple `authentication_failed` and `sshd` rule group alerts fired. Hydra successfully found the password `Lab@1234` after 6 failed attempts.

**Key rules fired:**
- Rule 5501 — PAM: Login session opened
- Rule 5502 — PAM: Login session closed
- Rule 5503 — PAM: User login failed
- Rule 5402 — Successful sudo to ROOT executed

**Analysis:** Wazuh's built-in PAM and sshd rules detected both the failed login attempts and the eventual successful login. In a real SOC, the pattern of multiple failures followed by a success from the same source IP is a textbook brute force indicator.

**MITRE:** T1110.001 — Brute Force: Password Guessing
**Verdict:** DETECTED

---

## Attack 3 — Create Local Admin User (T1136.001)

**Target:** Windows victim
**Tool:** net.exe from elevated PowerShell
**Commands:**
```powershell
net user hacker P@ssw0rd123! /add
net localgroup Administrators hacker /add
net user hacker
net user hacker /delete
```

**Wazuh result:** Detected at Level 12 (HIGH).

**Key rules fired:**
- Rule 60154 — Administrators Group Changed (Level 12)
- Rule 60111 — User account disabled or deleted (Level 8)
- Rule 60160 — Domain Users Group Changed (Level 5)
- Rule 60170 — Users Group Changed (Level 5)
- Rule 92039 — net.exe account discovery command initiated
- Rule 92033 — Discovery activity spawned via PowerShell
- Rule 92213 — Executable dropped in folder commonly used by malware (Level 15)

**Analysis:** This attack generated the richest detection of all 8 techniques. Wazuh caught the user creation, the privilege escalation to Administrators group, the discovery commands, AND the cleanup (deletion). Rule 60154 at Level 12 would trigger an immediate SOC escalation. The Level 15 alert for executable dropping is the highest severity Wazuh assigns.

**MITRE:** T1136.001 — Create Account: Local Account
**Verdict:** DETECTED — Level 12+15 (Critical)

---

## Attack 4 — Registry Run Key Persistence (T1547.001)

**Target:** Windows victim
**Tool:** reg.exe from elevated PowerShell
**Commands:**
```powershell
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /t REG_SZ /d "C:\Windows\Temp\malware.exe" /f
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate"
reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "WindowsUpdate" /f
```

**Wazuh result:** Detected at Level 10.

**Key rules fired:**
- Rule 92302 — Registry entry to be executed on next logon was modified using reg.exe (Level 6)
- Rule 92041 — Value added to registry key has Base64-like pattern (Level 10)
- Rule 92033 — Discovery activity spawned via PowerShell execution
- Rule 92217 — Executable dropped in Windows root folder (Level 6)

**Analysis:** Wazuh detected the registry modification through Sysmon Event ID 13 (RegistryEvent: Value Set). Rule 92302 specifically targets Run key modifications — exactly the persistence technique used. The Base64-like pattern detection was a bonus false positive that shows how sensitive the rules are.

**MITRE:** T1547.001 — Boot or Logon Autostart Execution: Registry Run Keys
**Verdict:** DETECTED — Level 10

---

## Attack 5 — Scheduled Task Persistence (T1053.005)

**Target:** Windows victim
**Tool:** schtasks.exe from elevated PowerShell
**Commands:**
```powershell
schtasks /create /tn "SystemHealthCheck" /tr "C:\Windows\Temp\payload.exe" /sc onlogon /ru SYSTEM /f
schtasks /query /tn "SystemHealthCheck"
schtasks /delete /tn "SystemHealthCheck" /f
```

**Wazuh result:** Detected via Sysmon process creation and command-line logging.

**Analysis:** Sysmon EID 1 captured the full command line of schtasks.exe including the task name, trigger, and executable path. Wazuh's process creation rules flagged the activity.

**MITRE:** T1053.005 — Scheduled Task/Job: Scheduled Task
**Verdict:** DETECTED

---

## Attack 6 — Encoded PowerShell Execution (T1059.001)

**Target:** Windows victim
**Tool:** PowerShell with -enc flag
**Commands:**
```powershell
powershell -enc dwBoAG8AYQBtAGkA
```
(Base64-encoded "whoami" — harmless but mimics attacker behavior)

**Wazuh result:** Detected via Sysmon process creation.

**Analysis:** Sysmon EID 1 captured the full command line including the `-enc` flag and the Base64 payload. This is how real malware (Emotet, Cobalt Strike, PowerShell Empire) obfuscates commands. The detection proves that Sysmon + Wazuh can see through encoded PowerShell execution — a critical capability for any SOC.

**MITRE:** T1059.001 — Command and Scripting Interpreter: PowerShell
**Verdict:** DETECTED

---

## Attack 7 — Disable Windows Defender (T1562.001)

**Target:** Windows victim
**Tool:** PowerShell Set-MpPreference
**Commands:**
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
# Re-enabled after testing:
Set-MpPreference -DisableRealtimeMonitoring $false
```

**Wazuh result:** Detected at Level 15 (CRITICAL).

**Key rules fired:**
- Rule 92213 — Executable file dropped in folder commonly used by malware (Level 15)
- Rule 92058 — Application Compatibility Database launched (Level 12)
- Rule 92027 — PowerShell process spawned PowerShell instance (Level 4)
- Rule 60642 — Software protection service scheduled successfully

**Analysis:** Disabling Defender triggered the highest severity alert Wazuh can produce (Level 15). The Application Compatibility Database launch at Level 12 is an additional indicator of defense evasion. In a real SOC, a Level 15 alert generates an immediate incident response. The fact that the defender-disable command was caught through its side effects (file drops, shim activity) rather than a direct "Defender disabled" rule shows the depth of Sysmon's behavioral monitoring.

**MITRE:** T1562.001 — Impair Defenses: Disable or Modify Tools
**Verdict:** DETECTED — Level 15 (Critical)

---

## Attack 8 — SUID Binary Creation (T1548.001)

**Target:** Ubuntu victim
**Tool:** chmod from bash
**Commands:**
```bash
sudo cp /usr/bin/find /tmp/find_backdoor
sudo chmod u+s /tmp/find_backdoor
ls -la /tmp/find_backdoor
sudo rm /tmp/find_backdoor
```

**Wazuh result:** Only sudo/PAM events detected. No specific SUID alert.

**Key rules fired:**
- Rule 5402 — Successful sudo to ROOT executed
- Rule 5501/5502 — PAM session opened/closed
- Rule 5503 — PAM: User login failed

**Analysis:** The default Wazuh ruleset and the Neo23x0 auditd rules don't include a specific detection for SUID bit modifications. While the sudo commands were logged, the actual privilege escalation technique (setting SUID on a binary) was invisible. This is a significant gap — an attacker who already has sudo could create a SUID backdoor and the SOC would miss it. Addressed in Phase 5 with a custom auditd + Wazuh rule.

**MITRE:** T1548.001 — Abuse Elevation Control Mechanism: Setuid and Setgid
**Verdict:** DETECTION GAP

---

## MITRE ATT&CK Coverage Matrix

| # | Technique | ID | Target | Detected | Severity | Key Rule |
|---|---|---|---|---|---|---|
| 1 | Network Service Scanning | T1046 | Both | NO | — | No rule exists |
| 2 | Brute Force: Password Guessing | T1110.001 | Ubuntu | YES | Medium | PAM auth_failed |
| 3 | Create Account: Local Account | T1136.001 | Windows | YES | **Level 12+15** | 60154 + 92213 |
| 4 | Registry Run Keys | T1547.001 | Windows | YES | Level 10 | 92302 + 92041 |
| 5 | Scheduled Task | T1053.005 | Windows | YES | Medium | Sysmon EID 1 |
| 6 | PowerShell Encoded Execution | T1059.001 | Windows | YES | Medium | Sysmon EID 1 |
| 7 | Disable Defender | T1562.001 | Windows | YES | **Level 15** | 92213 + 92058 |
| 8 | SUID Binary | T1548.001 | Ubuntu | NO | — | No rule exists |

**Detection rate: 6/8 (75%)**
**After Phase 5 custom rules: 8/8 (100%)**

---

## Key findings

1. **Windows detection is excellent.** Sysmon + SwiftOnSecurity config + Wazuh's built-in rules caught every Windows-targeted attack, including Level 15 critical alerts for defense evasion.

2. **Linux detection has gaps.** The default auditd rules detect authentication events well but miss kernel-level techniques like SUID manipulation and network-layer events like port scans.

3. **Highest-value detections:** Rules 60154 (Admin Group Changed, Level 12) and 92213 (Executable in malware folder, Level 15) would generate immediate SOC escalations in a production environment.

4. **Detection through side effects works.** Attack 7 (Disable Defender) wasn't caught by a "Defender disabled" rule — it was caught through behavioral side effects (file drops, shim databases, process spawning). This is how modern detection engineering works.

5. **The 2 gaps are addressable.** Both T1046 (port scan) and T1548.001 (SUID) can be filled with custom rules — see Phase 5.

---

## Lessons learned

- **Run attacks, don't assume coverage.** Before Phase 4, I assumed auditd with 354 rules would catch SUID modifications. Testing proved otherwise.
- **High severity != most dangerous.** The Level 15 alerts were impressive but the Level 12 "Admin Group Changed" is arguably more actionable — it's a clear, unambiguous indicator of compromise.
- **Cleanup gets detected too.** Deleting the hacker user (Rule 60111) and removing registry keys all generated alerts. An attacker trying to cover their tracks would create MORE noise, not less.
- **The gap analysis is the real deliverable.** Any tool can generate alerts. Knowing WHERE it fails and HOW to fix it is what separates a junior analyst from a detection engineer.
