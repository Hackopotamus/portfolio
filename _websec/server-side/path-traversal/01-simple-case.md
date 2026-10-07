---
title: "File path traversal, simple case"
kind: lab
category: "Server-side"
module: path-traversal
level: apprentice
order: 1
date: 2026-10-07
description: "Reading /etc/passwd through an unvalidated image filename parameter, with a remediation write-up and CVSS 3.1 score."
---

This lab contains a path traversal vulnerability in the way product images are displayed.

To solve the lab, we need to retrieve the contents of the `/etc/passwd` file.

## Solution Write-Up

There are two ways to solve this lab, but one also shows how Burp Suite allows us to validate and investigate the vulnerability in more detail than using the browser alone. To show this, let's first trigger the path traversal vulnerability solely using the browser and then move over to Burp Suite to see how its features allow us to inspect and manipulate the request.

### Manual - Browser Only

To start, we can select any item from the store to view the product. From there, we can inspect the product's image by right-clicking it and selecting `Open Image in New Tab`. This allows us to isolate the image URL being used to retrieve the image from the server.

We can now inspect the URL to see how the image is retrieved using the `image?filename=` parameter. This parameter is used to specify which image the application should load. As an attacker, we can manipulate this value and attempt to traverse the filesystem.

![Product image opened in a new tab, showing the filename parameter]({{ '/assets/img/websec/path-traversal/01-image-url.png' | relative_url }})

If we alter the URL to the below, we can trigger the vulnerability without using Burp Suite. Using `../../../`, we are traversing up the directory structure until we reach the filesystem root, allowing us to request `/etc/passwd` directly.

```text
https://<lab-id>.web-security-academy.net/image?filename=../../../etc/passwd
```

This allows us to complete the lab, as we can see below. However, we are unable to see the contents of `/etc/passwd` because the browser treats the response as an image.

This is fine for proving that we understand the concept and what is happening, but in a real engagement there will be no lab completion banner. We will need to provide evidence that demonstrates the vulnerability and can be presented to the client.

![Lab solved banner]({{ '/assets/img/websec/path-traversal/01-completed.png' | relative_url }})

### Burp - Request Tampering

With those limitations in mind, this is where Burp earns its place: it shows us the actual response, which gives us evidence we can present as a finding and lets us validate it properly.

We will reset the lab environment in order to start fresh and then go back to the store and select any product. Once we are on the product page, we can set FoxyProxy to our Burp proxy and then start Burp Intercept.

With Intercept active, we can use the same trick as last time and right-click the product image and select `Open Image in New Tab`. This will generate a new request, which we intercept in Burp's proxy. We can see the `image?filename=` parameter that we altered last time, and press `Ctrl + R` to send the request to the Repeater module.

![Intercepted image request in Burp Proxy]({{ '/assets/img/websec/path-traversal/01-proxy.png' | relative_url }})

By using the same path traversal string as before, we can edit the raw `GET` request and change the `image?filename=` parameter within the Repeater tab (see image below). This allows us to tamper with the request before it is sent to the server, as well as inspect the response once the request has been processed.

![Modified request in Burp Repeater]({{ '/assets/img/websec/path-traversal/01-repeater.png' | relative_url }})

After we send the newly edited request, we will complete the lab once again, but this time we can see the response containing the contents of `/etc/passwd`, allowing us to validate our findings.

This means we have completed the lab. However, this is not the end of our journey, as we can now use the above as a finding and demonstrate how we would report such a vulnerability.

## Vulnerability Remediation Write-Up

Now we can focus on how we would propose reporting such a vulnerability. This will include recommendations on how to remediate the issue, as well as scoring the potential risk associated with the finding.

We need to bear in mind that this is a lab environment, so we don't have the same context or impact assessment that we would have during a real engagement. However, we can still attempt to create a report that would provide useful information and assist with client remediation if this were presented as a finding during an assessment.

### Finding: Path Traversal in Image Retrieval Endpoint

**Severity:** High  
**CVSS 3.1 Score:** [7.5](https://www.first.org/cvss/calculator/3.1#CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N)  
**CVSS 3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`  
**Affected endpoint:** `GET /image?filename=`

**Description**

The web store application serves product images through the `/image` endpoint, which accepts a filename from the `filename` query parameter. This value is appended to a base directory and used to read a file from the server's filesystem without any validation. By supplying directory traversal sequences (`../`), an attacker can escape the intended images directory and read arbitrary files that the web application's user account can access.

**Proof of concept**

The following request returned the contents of `/etc/passwd`:

```http
GET /image?filename=../../../etc/passwd
```

The response contained the system's user account list, confirming that files outside the web root are readable.

![Response containing /etc/passwd]({{ '/assets/img/websec/path-traversal/01-poc.png' | relative_url }})

**Impact**

An unauthenticated attacker can read any file the application process has permission to access. Depending on the environment, this may include application source code, configuration files containing database credentials or API keys, and SSH keys. Exposure of these could lead to further compromise, such as database access or lateral movement.

**Remediation**

_Primary fix (depending on use case):_

1. **Stop accepting filenames from the user.** Reference images by an identifier (e.g. `?id=21`) that the server maps to a file from a fixed, server-side list. Requests for any unknown identifier should return a 404. Alternatively, serve images as static content directly from the web server or a CDN, removing the need for an application handler to read from the filesystem.
2. **If a filename must be accepted,** resolve the full canonical path and confirm it remains within the intended directory before opening the file. As the backend technology is unknown, the example below is an illustrative Python implementation.

```python
import os

base = os.path.realpath("/var/www/images")
target = os.path.realpath(os.path.join(base, filename))
if not target.startswith(base + os.sep):
    abort(404)
```

Input should also be restricted to an expected format (e.g. alphanumeric characters and a single permitted extension), rejecting path separators and null bytes.

_Note:_ Removing or filtering `../` sequences is not an adequate fix. Such filters are routinely bypassed using absolute paths, nested sequences (`....//`), URL encoding, or null bytes.

_Defence in depth:_

- Run the application under a low-privileged account with filesystem access restricted to the directories it needs.
- Store secrets outside files readable by the application where possible, e.g. in a secrets manager.
- Log and alert on requests containing traversal patterns (`../`, `%2e%2e`, `%2f`, `/etc/passwd`) to detect exploitation attempts.

**References**

- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [PortSwigger Web Security Academy: Path traversal](https://portswigger.net/web-security/file-path-traversal)
- [CWE-22](https://cwe.mitre.org/data/definitions/22.html)
