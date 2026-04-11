# RDP Security Hardening — Windows Server

> **Audience:** Windows Server administrators (on-premises)  
> **Goal:** Secure Remote Desktop from basic port hardening to audit logging  
> **OS:** Windows Server 2016 / 2019 / 2022

---

## Table of Contents

1. [RDP Port Change](#1-rdp-port-change-3389--custom)
2. [Network Level Authentication (NLA)](#2-network-level-authentication-nla)
3. [Account Lockout Policy](#3-account-lockout-policy)
4. [IP Whitelisting](#4-ip-whitelisting)
5. [TLS Encryption Level](#5-tls-encryption-level)
6. [Audit Logging](#6-audit-logging)
7. [Final Security Checklist](#7-final-security-checklist)
8. [One-Shot Hardening Script](#8-one-shot-hardening-script)

---

## Why Harden RDP?

Remote Desktop Protocol (RDP) is one of the most targeted attack surfaces on Windows servers. The default port `3389` is continuously scanned by automated bots across the internet. Without proper hardening, your server is exposed to:

- **Brute force attacks** — automated tools trying thousands of username/password combinations
- **Credential stuffing** — using leaked credentials from other breaches
- **Man-in-the-middle attacks** — intercepting unencrypted or weakly encrypted RDP sessions
- **BlueKeep / DejaBlue** — RDP vulnerabilities that allow unauthenticated remote code execution

This guide walks through layered security controls — each step adds another layer of protection.

---

## 1. RDP Port Change (3389 → Custom)

The default RDP port `3389` is the first thing attackers probe. Changing it to a non-standard port significantly reduces automated scan noise and opportunistic attacks.

> ⚠️ **Critical Warning:** Keep an existing RDP session open while making these changes. Test the new port before closing your current session — otherwise you risk locking yourself out.

### Step 1 — Change the port in the Registry

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" ^
  /v PortNumber /t REG_DWORD /d 52389 /f
```

Or via PowerShell:

```powershell
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
  -Name "PortNumber" -Value 52389
```

> **Port selection:** Use any port in the range `49152–65535` (private/dynamic range). Avoid well-known ports like `8080`, `443`, or `8443` as those are also commonly scanned.

### Step 2 — Add a firewall rule for the new port

```powershell
# Allow inbound traffic on the new RDP port
New-NetFirewallRule -DisplayName "RDP-Custom" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 52389 `
  -Action Allow

# Disable the default RDP firewall rule (port 3389)
Disable-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In)"
```

### Step 3 — Restart the RDP service

```powershell
Restart-Service TermService -Force
```

### Step 4 — Verify the new port is listening

```powershell
netstat -ano | findstr :52389
```

**Expected output:**
```
TCP    0.0.0.0:52389    0.0.0.0:0    LISTENING    <PID>
```

If there is no output, the service did not pick up the change. Re-check the registry value and restart the service again.

### Connecting with a Custom Port

| RDP Client | Connection Format |
|---|---|
| Windows `mstsc` | `192.168.1.10:52389` |
| Mac — Microsoft Remote Desktop | Host: `192.168.1.10` → Port field: `52389` (separate field) |
| Linux `rdesktop` | `rdesktop 192.168.1.10:52389` |
| Linux `xfreerdp` | `xfreerdp /v:192.168.1.10:52389` |

---

## 2. Network Level Authentication (NLA)

Without NLA, an attacker can reach the Windows login screen without any prior authentication — making brute force attacks easier and exposing the server to pre-authentication exploits.

With NLA enabled, the connecting user must authenticate **before** a full RDP session is established. This means:
- Unauthenticated users never reach the desktop
- The server's resources are not consumed by unauthorized session setup
- Pre-auth vulnerabilities like BlueKeep are mitigated

### Enable NLA via PowerShell

```powershell
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
  -Name "UserAuthentication" -Value 1
```

### Enable NLA via GUI

```
Right-click "This PC" → Properties
→ Remote settings
→ Remote Desktop section
→ Select: "Allow connections only from computers running Remote Desktop
           with Network Level Authentication (recommended)"
```

### Verify NLA is enabled

```powershell
Get-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
  -Name "UserAuthentication"
```

**Expected output:** `UserAuthentication : 1`

> **Compatibility note:** NLA is supported on Windows 7 and later, and on the Microsoft Remote Desktop app for macOS and iOS. Older clients (Windows XP) will be unable to connect — which is generally the desired behavior.

---

## 3. Account Lockout Policy

Account lockout prevents brute force attacks by temporarily disabling an account after a defined number of failed login attempts.

### Recommended Settings

| Policy Setting | Recommended Value | Reason |
|---|---|---|
| Account lockout threshold | **5 invalid attempts** | Stops brute force while allowing for typos |
| Account lockout duration | **30 minutes** | Long enough to deter attackers, short enough for legitimate users |
| Reset lockout counter after | **15 minutes** | Resets the failed attempt counter |

### Configure via Command Line

```cmd
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:15
```

### Configure via Local Security Policy (GUI)

```
Run: secpol.msc
→ Account Policies
→ Account Lockout Policy
→ Set the three values above
```

### Verify the policy

```cmd
net accounts
```

Look for the `Lockout threshold`, `Lockout duration`, and `Lockout observation window` lines in the output.

> ⚠️ **Warning:** The built-in `Administrator` account can also be locked out. Either keep a secondary admin account as a backup, or ensure the primary Administrator account has a sufficiently strong and unique password.

---

## 4. IP Whitelisting

Restrict RDP access so that only known, authorized IP addresses or subnets can connect. Any connection attempt from an unlisted IP is silently dropped by the firewall.

This is one of the most effective controls available — even if an attacker knows your port and has valid credentials, they cannot connect from an unauthorized network.

### Step 1 — Restrict the existing firewall rule to specific IPs

```powershell
# Replace with your actual authorized IPs or subnets
Set-NetFirewallRule -DisplayName "RDP-Custom" `
  -RemoteAddress 192.168.62.0/24, 10.10.1.50
```

You can specify:
- A single IP: `10.10.1.50`
- A subnet: `192.168.62.0/24`
- Multiple entries separated by commas

### Step 2 — Add a catch-all block rule for everything else

```powershell
New-NetFirewallRule -DisplayName "RDP-Block-All-Others" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 52389 `
  -Action Block `
  -RemoteAddress Any
```

> **How this works:** Windows Firewall evaluates rules in priority order. A specific Allow rule (with a matching source IP) takes precedence over a general Block rule. Authorized IPs are allowed through while everything else is blocked.

### Step 3 — Verify the whitelist is applied

```powershell
Get-NetFirewallRule -DisplayName "RDP-Custom" | Get-NetFirewallAddressFilter
```

Check that the `RemoteAddress` field shows your intended IPs/subnets.

---

## 5. TLS Encryption Level

By default, Windows RDP uses "Negotiate" mode — which can fall back to weaker encryption if the client requests it. Forcing SSL/TLS ensures all sessions are encrypted with a strong cipher regardless of client settings.

### SecurityLayer Values

| Value | Mode | Description |
|---|---|---|
| `0` | RDP | Legacy RDP encryption only — **avoid** |
| `1` | Negotiate | Server accepts client's preferred method — default, not ideal |
| `2` | SSL (TLS) | Forces TLS — **recommended** |

### MinEncryptionLevel Values

| Value | Level | Description |
|---|---|---|
| `1` | Low | Only outgoing data is encrypted |
| `2` | Client-compatible | Encrypts to the highest level the client supports |
| `3` | High | 128-bit encryption minimum — **recommended** |
| `4` | FIPS compliant | Enforces FIPS 140-1 validated algorithms |

### Apply the recommended settings

```powershell
$RDPPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp"

# Force TLS (SSL) security layer
Set-ItemProperty -Path $RDPPath -Name "SecurityLayer"      -Value 2

# Enforce high (128-bit minimum) encryption
Set-ItemProperty -Path $RDPPath -Name "MinEncryptionLevel" -Value 3
```

### Restart the service to apply

```powershell
Restart-Service TermService -Force
```

### Verify the settings

```powershell
$RDPPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp"
Get-ItemProperty -Path $RDPPath | Select-Object SecurityLayer, MinEncryptionLevel
```

**Expected output:**
```
SecurityLayer      MinEncryptionLevel
-------------      ------------------
            2                       3
```

---

## 6. Audit Logging

Enable logon auditing to track all successful and failed RDP connection attempts. This is essential for detecting brute force attacks, unauthorized access, and suspicious login patterns.

### Enable Logon Event Auditing

```cmd
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
```

Verify it is active:

```cmd
auditpol /get /subcategory:"Logon"
```

### Query Failed Login Attempts (Event ID 4625)

```powershell
Get-WinEvent -FilterHashtable @{
  LogName = 'Security'
  Id      = 4625
} | Select-Object TimeCreated, Message | Select-Object -First 20
```

A high volume of Event ID `4625` from a single IP is a strong indicator of an active brute force attack.

### Query Successful RDP Sessions (Event ID 4624, Logon Type 10)

```powershell
Get-WinEvent -FilterHashtable @{
  LogName = 'Security'
  Id      = 4624
} | Where-Object { $_.Message -like "*Logon Type:*10*" } `
  | Select-Object TimeCreated, Message `
  | Select-Object -First 10
```

> **Logon Type 10** indicates a Remote Interactive logon — i.e., an RDP session.

### RDP-Relevant Event IDs Reference

| Event ID | Description |
|---|---|
| `4624` | Successful logon |
| `4625` | Failed logon attempt |
| `4634` | Logoff |
| `4648` | Logon attempted with explicit credentials |
| `4778` | RDP session reconnected |
| `4779` | RDP session disconnected |

### Recommended Monitoring Practice

Review the Security event log regularly — especially Event ID `4625`. If you see repeated failures from an IP that is not in your whitelist, investigate immediately. Consider setting up a scheduled task or a SIEM alert for bulk failures.

---

## 7. Final Security Checklist

Use this checklist to verify your hardening is complete before considering the server production-ready.

- [ ] RDP port changed from `3389` to a custom port (e.g. `52389`)
- [ ] Windows Firewall rule created for the new port — old `3389` rule disabled
- [ ] NLA enabled — `UserAuthentication = 1` confirmed in registry
- [ ] Account lockout configured — threshold: 5 attempts, duration: 30 minutes
- [ ] IP whitelisting applied — only authorized IPs/subnets can reach the RDP port
- [ ] TLS encryption enforced — `SecurityLayer = 2`, `MinEncryptionLevel = 3`
- [ ] Audit logging enabled — logon success and failure events being recorded
- [ ] Strong Administrator password in use (12+ characters, mixed case, numbers, symbols)
- [ ] Unnecessary or default user accounts reviewed and disabled
- [ ] Event Log review scheduled on a regular basis

---

## 8. One-Shot Hardening Script

Run this script as Administrator to apply all settings at once. Review and adjust the `$CustomPort` and `$AllowedIPs` variables before running.

```powershell
# ============================================================
# RDP Security Hardening Script
# Author: Anuj — Subharti Hospital IT
# Run as: Administrator
# ============================================================

# ---- CONFIGURATION ----
$CustomPort = 52389
$AllowedIPs = @("192.168.62.0/24")   # Add your authorized IPs/subnets here
# -----------------------

$RDPPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp"

Write-Host "`n[RDP Hardening Script]" -ForegroundColor Cyan
Write-Host "================================" -ForegroundColor Cyan

# 1. Change RDP Port
Write-Host "`n[1/6] Changing RDP port to $CustomPort..." -ForegroundColor Yellow
Set-ItemProperty -Path $RDPPath -Name "PortNumber" -Value $CustomPort
Write-Host "      Done." -ForegroundColor Green

# 2. Enable NLA
Write-Host "`n[2/6] Enabling Network Level Authentication..." -ForegroundColor Yellow
Set-ItemProperty -Path $RDPPath -Name "UserAuthentication" -Value 1
Write-Host "      Done." -ForegroundColor Green

# 3. Set TLS Encryption
Write-Host "`n[3/6] Setting SecurityLayer to TLS and encryption to High..." -ForegroundColor Yellow
Set-ItemProperty -Path $RDPPath -Name "SecurityLayer"      -Value 2
Set-ItemProperty -Path $RDPPath -Name "MinEncryptionLevel" -Value 3
Write-Host "      Done." -ForegroundColor Green

# 4. Configure Firewall Rules
Write-Host "`n[4/6] Configuring firewall rules..." -ForegroundColor Yellow
New-NetFirewallRule -DisplayName "RDP-Custom" `
  -Direction Inbound -Protocol TCP `
  -LocalPort $CustomPort -Action Allow `
  -RemoteAddress $AllowedIPs `
  -ErrorAction SilentlyContinue | Out-Null

New-NetFirewallRule -DisplayName "RDP-Block-All-Others" `
  -Direction Inbound -Protocol TCP `
  -LocalPort $CustomPort -Action Block `
  -RemoteAddress Any `
  -ErrorAction SilentlyContinue | Out-Null

Disable-NetFirewallRule -DisplayName "Remote Desktop - User Mode (TCP-In)" `
  -ErrorAction SilentlyContinue
Write-Host "      Done." -ForegroundColor Green

# 5. Account Lockout Policy
Write-Host "`n[5/6] Applying account lockout policy..." -ForegroundColor Yellow
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:15 | Out-Null
Write-Host "      Done." -ForegroundColor Green

# 6. Enable Audit Logging
Write-Host "`n[6/6] Enabling logon audit logging..." -ForegroundColor Yellow
auditpol /set /subcategory:"Logon" /success:enable /failure:enable | Out-Null
Write-Host "      Done." -ForegroundColor Green

# Restart RDP Service
Write-Host "`n[*] Restarting RDP service to apply changes..." -ForegroundColor Yellow
Restart-Service TermService -Force
Write-Host "    Service restarted." -ForegroundColor Green

# Summary
Write-Host "`n================================" -ForegroundColor Cyan
Write-Host "  Hardening Complete!" -ForegroundColor Green
Write-Host "  New RDP Port : $CustomPort" -ForegroundColor Green
Write-Host "  Allowed IPs  : $($AllowedIPs -join ', ')" -ForegroundColor Green
Write-Host "`n  IMPORTANT: Test your connection on port $CustomPort" -ForegroundColor Yellow
Write-Host "  before closing this session!" -ForegroundColor Yellow
Write-Host "================================`n" -ForegroundColor Cyan
```

### Usage

1. Open PowerShell as Administrator
2. Edit `$CustomPort` and `$AllowedIPs` at the top of the script
3. Save as `rdp-harden.ps1`
4. Run:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope Process
.\rdp-harden.ps1
```

---

## Additional Recommendations

Beyond the steps in this guide, consider these controls for a production environment:

| Control | Description |
|---|---|
| **VPN Gateway** | Place RDP behind a VPN — do not expose it directly to the internet even with a custom port |
| **Jump Server / Bastion Host** | Route all RDP access through a dedicated hardened intermediate host |
| **RD Gateway** | Tunnels RDP over HTTPS (port 443) — useful for remote access without a full VPN |
| **IPsec Connection Security Rules** | Enforce IPsec-authenticated connections via Windows Defender Firewall with Advanced Security |
| **Regular Patching** | Keep Windows Server patched — RDP vulnerabilities (BlueKeep, DejaBlue) are patched via Windows Update |
| **Disable RDP when not needed** | If RDP is used only occasionally, disable it and enable only when required |

---

## References

- [Microsoft Docs — Remote Desktop Services Security](https://docs.microsoft.com/en-us/windows-server/remote/remote-desktop-services/rds-security)
- [CIS Benchmark — Windows Server 2019](https://www.cisecurity.org/benchmark/microsoft_windows_server)
- [NIST SP 800-46 Rev 2 — Guide to Enterprise Telework and Remote Access Security](https://csrc.nist.gov/publications/detail/sp/800-46/rev-2/final)
- [CVE-2019-0708 — BlueKeep](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2019-0708)

---

*Maintained by Anuj*  
*Last updated: 2026*
