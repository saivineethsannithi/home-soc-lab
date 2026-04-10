# Phase 1 — Lab Infrastructure Setup

Building the virtualized environment that hosts the whole lab.

## Goals
1. Four VMs running on an isolated private network
2. Full network connectivity between victims, attacker, and SIEM
3. Clean snapshots for every VM as rollback points

## VM Specifications

| VM | OS | RAM | CPU | Disk | Role |
|---|---|---|---|---|---|
| `SOC-Lab-Wazuh-SIEM` | Amazon Linux 2023 | 4 GB | 2 | 50 GB | SIEM / Indexer / Dashboard |
| `SOC-Lab-Win11-Victim` | Windows 11 Enterprise Eval | 4 GB | 2 | 50 GB | Victim endpoint |
| `SOC-Lab-Ubuntu-Victim` | Ubuntu Server 22.04 LTS | 1.5 GB | 1 | 25 GB | Victim server |
| `SOC-Lab-Kali-Attacker` | Kali Linux 2024.x | 2.5 GB | 2 | 40 GB | Attack simulation |

**Host:** Intel i7-13650HX, 16 GB RAM, Windows 11

## Network Design
- **Type:** VirtualBox NAT Network
- **Name:** `SOCLab`
- **Subnet:** `10.0.10.0/24`
- **DHCP:** Enabled
- **IPv6:** Disabled

This gives all four VMs a private LAN where they can communicate with each
other and reach the internet for updates, but the outside world cannot
reach them. Host access to Wazuh dashboard is provided via port forwarding.

## Troubleshooting Log (real issues hit)

### Issue 1: Black screen on Windows 11 VM first boot
**Cause:** Hyper-V was enabled on the host laptop, stealing virtualization
from VirtualBox. VM booted in degraded compatibility mode and hung in EFI.

**Fix:**
```powershell
bcdedit /set hypervisorlaunchtype off
DISM /Online /Disable-Feature:Microsoft-Hyper-V-All /NoRestart
```
Then uncheck Hyper-V, Virtual Machine Platform, Windows Hypervisor Platform,
WSL, Windows Sandbox in "Turn Windows features on or off" and reboot the host.

### Issue 2: Windows 11 installer refused install (TPM/Secure Boot check)
**Fix:** Disabled EFI/Secure Boot/TPM in VM settings (not needed for a lab),
used the documented registry bypass during setup:
- `HKLM\SYSTEM\Setup\LabConfig\BypassTPMCheck = 1`
- `HKLM\SYSTEM\Setup\LabConfig\BypassSecureBootCheck = 1`
- `HKLM\SYSTEM\Setup\LabConfig\BypassRAMCheck = 1`

### Issue 3: Ubuntu VM kept re-launching installer on boot
**Cause:** Install ISO was still attached to the virtual CD drive after
install completed, and boot order had Optical above Hard Disk.

**Fix:** Settings → Storage → select the ISO → "Remove Disk from Virtual
Drive". Always eject install media after install. (Same lesson as real
hardware.)

### Issue 4: 100% packet loss pinging between VMs
**Cause:** Guessed IP addresses instead of verifying with `ip a` / `ipconfig`.

**Fix:** Always run `ip a` (Linux) or `ipconfig` (Windows) on each VM to get
the real IP before testing connectivity. This is the same discipline a SOC
analyst applies during investigations — never assume, always verify.

## Verification
All VMs on SOCLab can reach each other:
- Windows → Ubuntu: ping reply ✅
- Windows firewall allows ICMP Echo Request via `New-NetFirewallRule` (run
  from elevated PowerShell, learned to verify title bar says "Administrator:")

## Snapshots
Each VM has a `phase1-clean-networked` snapshot for rollback.
