# Phase 3 Closeout Screenshots — auditd on Ubuntu

Visual evidence of the Linux telemetry deployment.

## `auditd-package-install.png`
`apt install auditd audispd-plugins` output. Shows the package install
pulling in `libauparse0t64`, `auditd`, and `audispd-plugins` from the
Ubuntu noble-updates repo, creating the systemd service unit, and
processing triggers cleanly. Ends with `Running kernel seems to be
up-to-date.` — proof of a clean install.

## `auditd-systemctl-enable.png`
`sudo systemctl enable --now auditd` — synchronizing state of the
auditd service with the SysV script and enabling it for autostart.

## `auditd-neo23x0-wget.png`
`wget` pulling the Neo23x0 community auditd ruleset directly from
`raw.githubusercontent.com`. 31,963 bytes downloaded in 0.007s — about
~400 lines of community-tested audit rules covering Linux-relevant
MITRE ATT&CK techniques.

## `auditd-test-events.png`
Discovery activity simulation triggered after auditd was loaded:
- `id` showing UID/GID/groups (analyst, sudo, lxd, cdrom, dip, plugdev)
- `uname -a` showing kernel `6.8.0-107-generic` on Ubuntu 24.04
- `ps aux | head` showing running processes
- `ss -tulpn` showing listening ports including SSH on `0.0.0.0:22`
- `sudo touch /etc/passwd` triggering a sensitive file modification event

Each command represents an attacker discovery technique (MITRE TA0007)
that auditd is now configured to capture.

## `wazuh-ubuntu-421-events.png`
Wazuh Threat Hunting dashboard filtered to `agent.id: 002` (ubuntu-victim):
- **Total events:** 421
- **Level 12+ alerts:** 0 (clean baseline)
- **Authentication failures:** 6
- **Authentication successes:** 22

Top 10 alert groups visible: sca, syslog, pam, authentication_success,
sudo, dpkg, apparmor, config_changed, local, authentication_failed.

This is multi-source Linux telemetry flowing into the same SIEM as the
Windows side — Phase 3 is complete.
