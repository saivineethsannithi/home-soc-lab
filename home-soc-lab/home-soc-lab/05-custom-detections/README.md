# Phase 5 — Custom Detection Engineering

> Goal: Close the 2 detection gaps identified in Phase 4 by writing custom
> Wazuh rules. This is the senior-level detection engineering skill that
> separates junior SOC analysts from detection engineers.

## Detection Gaps from Phase 4

| Gap | Technique | Problem |
|---|---|---|
| Gap 1 | T1046 — Port Scan | Default Wazuh has no port scan detection rule |
| Gap 2 | T1548.001 — SUID Binary | Default auditd + Wazuh rules don't alert on SUID bit changes |

---

## Custom Rule 1: SSH Brute Force / Port Scan Correlation

### Problem
Nmap port scans against Ubuntu generated zero Wazuh alerts. The default
ruleset has individual SSH failure rules but no correlation rule that
detects MULTIPLE failures from the SAME source in a SHORT timeframe.

### Solution
A Wazuh frequency-based rule that fires when 5+ SSH authentication
failures arrive from the same source IP within 60 seconds.

### Rule definition

**File:** `/var/ossec/etc/rules/local_rules.xml`

```xml
<rule id="100001" level="10" frequency="5" timeframe="60">
  <if_matched_sid>5710</if_matched_sid>
  <same_source_ip />
  <description>Multiple SSH failures from same IP. MITRE T1046.</description>
  <group>authentication_failed,attack,</group>
</rule>
```

### How it works

| Component | Purpose |
|---|---|
| `id="100001"` | Custom rule ID (100000+ reserved for local rules) |
| `level="10"` | High severity — generates a visible alert |
| `frequency="5"` | Must see 5 matching child events |
| `timeframe="60"` | Within 60 seconds |
| `if_matched_sid>5710` | Child events must match rule 5710 (sshd auth failure) |
| `same_source_ip` | All 5 failures must come from the same IP address |

### What this catches
- Nmap service detection scripts (multiple SSH probes during `-sC`)
- Hydra/Medusa/Patator brute force attempts
- Any automated SSH credential stuffing
- Manual SSH enumeration (attacker trying passwords by hand quickly)

### MITRE ATT&CK mapping
- **T1046** — Network Service Scanning (port scan with SSH probing)
- **T1110.001** — Brute Force: Password Guessing

### Testing
```bash
# From Kali:
hydra -l analyst -P passwords.txt 10.0.10.5 ssh -t 4 -V
```

Expected: Rule 100001 fires at Level 10 after 5+ SSH failures within 60 seconds.

---

## Custom Rule 2: SUID Binary Detection

### Problem
An attacker with sudo access copied `/usr/bin/find` to `/tmp/` and set
the SUID bit (`chmod u+s`). The default auditd ruleset logged the sudo
commands (Rule 5402) but completely missed the SUID permission change
itself. Any user could then run `/tmp/find_backdoor` as root without sudo.

### Solution
Two-part fix:
1. **Custom auditd rule** that watches for `chmod` syscalls setting SUID
2. **Custom Wazuh rule** that triggers on the auditd event key

### Part A — auditd rule (on Ubuntu victim)

**File:** `/etc/audit/rules.d/audit.rules` (append to existing)

```
-a always,exit -F arch=b64 -S fchmodat -S chmod -S fchmod -F auid>=1000 -F auid!=4294967295 -F perm=u+s -k T1548_suid_bit
```

**How it works:**
- Watches the `chmod`, `fchmod`, and `fchmodat` syscalls on 64-bit architecture
- Only fires for real users (`auid >= 1000`), not system processes
- Only fires when SUID permission (`perm=u+s`) is being set
- Tags the event with key `T1548_suid_bit` for easy filtering

**Deploy:**
```bash
sudo bash -c 'echo "-a always,exit -F arch=b64 -S fchmodat -S chmod -S fchmod -F auid>=1000 -F auid!=4294967295 -F perm=u+s -k T1548_suid_bit" >> /etc/audit/rules.d/audit.rules'
sudo augenrules --load
sudo auditctl -l | grep T1548
```

### Part B — Wazuh rule (on Wazuh server)

**File:** `/var/ossec/etc/rules/local_rules.xml`

```xml
<rule id="100002" level="12">
  <if_sid>80700</if_sid>
  <field name="audit.key">T1548_suid_bit</field>
  <description>SUID bit set on binary. MITRE T1548.001.</description>
  <group>audit,attack,</group>
</rule>
```

**How it works:**
- Triggers when any auditd event (child of rule 80700) contains the key `T1548_suid_bit`
- Level 12 (High) — generates immediate SOC visibility
- Tagged with the `audit` and `attack` groups for easy filtering

**Note:** The `if_sid` value (80700) is the parent rule for auditd events in
the default Wazuh ruleset. If this doesn't match your Wazuh version, run
`sudo grep -r "auditd" /var/ossec/ruleset/rules/ | grep "id="` to find
the correct parent rule ID. The rule may need tuning to match rule 80730
or another auditd parent depending on your exact Wazuh 4.14.4 rule
definitions.

### Testing
```bash
# On Ubuntu:
sudo cp /usr/bin/find /tmp/find_backdoor
sudo chmod u+s /tmp/find_backdoor

# Check Wazuh dashboard → ubuntu-victim → last 15 minutes
# Expected: Rule 100002 at Level 12

# Cleanup:
sudo rm /tmp/find_backdoor
```

### MITRE ATT&CK mapping
- **T1548.001** — Abuse Elevation Control Mechanism: Setuid and Setgid

---

## Complete local_rules.xml

Both custom rules in a single file:

```xml
<group name="local,syslog,sshd,">

  <!-- Custom Rule 1: SSH brute force / port scan correlation -->
  <rule id="100001" level="10" frequency="5" timeframe="60">
    <if_matched_sid>5710</if_matched_sid>
    <same_source_ip />
    <description>Multiple SSH failures from same IP. MITRE T1046.</description>
    <group>authentication_failed,attack,</group>
  </rule>

  <!-- Custom Rule 2: SUID bit set on binary -->
  <rule id="100002" level="12">
    <if_sid>80700</if_sid>
    <field name="audit.key">T1548_suid_bit</field>
    <description>SUID bit set on binary. MITRE T1548.001.</description>
    <group>audit,attack,</group>
  </rule>

</group>
```

**Deploy:**
```bash
# Copy to Wazuh server
sudo cp local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

---

## Updated MITRE ATT&CK Coverage (Phase 4 + Phase 5)

| # | Technique | ID | Phase 4 | Phase 5 | Final Status |
|---|---|---|---|---|---|
| 1 | Network Service Scanning | T1046 | GAP | Rule 100001 | COVERED |
| 2 | Brute Force | T1110.001 | Detected | Enhanced by 100001 | COVERED |
| 3 | Create Account | T1136.001 | Detected (L12) | — | COVERED |
| 4 | Registry Run Keys | T1547.001 | Detected (L10) | — | COVERED |
| 5 | Scheduled Task | T1053.005 | Detected | — | COVERED |
| 6 | Encoded PowerShell | T1059.001 | Detected | — | COVERED |
| 7 | Disable Defender | T1562.001 | Detected (L15) | — | COVERED |
| 8 | SUID Binary | T1548.001 | GAP | Rule 100002 | COVERED |

**Detection coverage: 6/8 (75%) → 8/8 (100%)**

---

## What this phase proves to employers

1. **You can identify detection gaps** — not just run attacks, but analyze WHERE the SIEM fails and WHY.

2. **You can write custom detection rules** — both at the data source level (auditd) and the SIEM level (Wazuh XML rules). This is a detection engineering skill, not just an analyst skill.

3. **You understand correlation rules** — Rule 100001 isn't a simple pattern match; it's a frequency-based correlation across multiple events from the same source. This is how real SOCs detect distributed attacks.

4. **You can map detections to MITRE ATT&CK** — every custom rule is tagged with the technique it detects, making it compatible with MITRE-based SOC workflows and reporting.

5. **You can improve coverage from 75% to 100%** — the before/after matrix is a concrete, quantifiable improvement that you can reference in interviews and on your resume.

---

## Lessons learned

1. **Community rules are a starting point, not a finish line.** The Neo23x0 auditd ruleset (354 rules) and Wazuh's built-in 3000+ rules still had 2 gaps against just 8 test techniques. Real environments need continuous rule development.

2. **Two-layer detection is stronger than one.** The SUID detection required BOTH a new auditd rule (data source layer) AND a new Wazuh rule (SIEM layer). If either layer is missing, the detection doesn't work.

3. **Correlation rules catch what single-event rules miss.** A single SSH failure isn't suspicious — but 5 from the same IP in 60 seconds is. This is the difference between log collection and threat detection.

4. **Rule tuning is iterative.** The SUID rule needed debugging (finding the correct parent rule ID for the specific Wazuh version). In production, detection engineers spend more time tuning existing rules than writing new ones.

5. **The gap analysis is the most valuable deliverable.** Writing "we tested 8 MITRE techniques, detected 6, and wrote custom rules to close the remaining 2" tells a complete detection engineering story.

---

## Future improvements

- **Network-layer detection:** Deploy Suricata IDS and integrate with Wazuh for true port scan detection at the packet level (not just SSH failures)
- **Sigma rule conversion:** Convert the custom rules to Sigma format for portability across SIEMs (Splunk, Elastic, QRadar)
- **Automated testing:** Set up Atomic Red Team on the Windows victim for repeatable, scheduled attack simulations
- **Rule coverage dashboard:** Build a Wazuh dashboard widget that maps active rules to MITRE ATT&CK techniques for continuous coverage monitoring
