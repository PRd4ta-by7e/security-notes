# Nmap Reference

A working reference for network scanning and enumeration with Nmap, built from hands-on lab practice.

## Table of Contents

1. [Host Discovery](#1-host-discovery)
2. [Port States](#2-port-states)
3. [Scan Types](#3-scan-types)
4. [Port Selection](#4-port-selection)
5. [Service & Version Detection](#5-service--version-detection)
6. [OS Detection](#6-os-detection)
7. [NSE Scripting](#7-nse-scripting)
8. [Timing & Performance](#8-timing--performance)
9. [Firewall / IDS Evasion](#9-firewall--ids-evasion)
10. [Output Formats](#10-output-formats)
11. [Go-To Recipe Combos](#11-go-to-recipe-combos)
12. [Field Notes](#12-field-notes)

---

## 1. Host Discovery

Determine what's alive before scanning ports.

```
nmap -sn 10.10.10.0/24          # ping sweep, no port scan
nmap -sn -PR 10.10.10.0/24      # ARP ping (local subnet, fast/reliable)
nmap -PS22,80,443 10.10.10.10   # TCP SYN ping to specific ports
nmap -PA80,443 10.10.10.10      # TCP ACK ping
nmap -PU53,161 10.10.10.10      # UDP ping
nmap -PE 10.10.10.10            # ICMP echo
nmap -Pn 10.10.10.10            # skip host discovery, treat as up (needed when ICMP is blocked)
```

## 2. Port States

| State | Meaning |
|---|---|
| `open` | Application actively accepting connections |
| `closed` | Port reachable, no application listening |
| `filtered` | Firewall/filter is blocking, Nmap can't tell open vs closed |
| `unfiltered` | Reachable but state undetermined (ACK scan result) |
| `open\|filtered` | No response, could be either (common with UDP) |
| `closed\|filtered` | Can't determine between closed and filtered |

## 3. Scan Types

```
nmap -sS target      # SYN scan (default w/ root), stealthier, doesn't complete handshake
nmap -sT target      # TCP connect scan, full handshake, no root needed, noisier
nmap -sU target      # UDP scan, slow, often needs -sV to confirm real state
nmap -sA target      # ACK scan, maps firewall rules, not port state
nmap -sW target      # Window scan, variant of ACK using TCP window field
nmap -sM target      # Maimon scan, FIN/ACK probe, evades some stateless filters
nmap -sN target      # Null scan, no flags set
nmap -sF target      # FIN scan
nmap -sX target      # Xmas scan, FIN, PSH, URG flags set
```

Null/FIN/Xmas rely on RFC 793 behavior and generally only work reliably against non-Windows stacks.

## 4. Port Selection

```
nmap -p 80 target                  # single port
nmap -p 1-1000 target              # range
nmap -p- target                    # all 65535 ports
nmap -p U:53,161,T:21-25,80 target # mixed UDP/TCP
nmap --top-ports 100 target        # top N most common ports
nmap -F target                     # fast mode, top 100 ports only
```

## 5. Service & Version Detection

```
nmap -sV target                # probe open ports for service/version info
nmap -sV --version-intensity 9 target   # more thorough probing (0-9, higher = slower/more accurate)
nmap -sV --version-light target         # faster, less thorough
```

## 6. OS Detection

```
nmap -O target                 # OS fingerprinting via TCP/IP stack behavior
nmap -O --osscan-guess target  # more aggressive guessing when match is ambiguous
```

Requires at least one open and one closed port for reliable results.

## 7. NSE Scripting

```
nmap -sC target                          # run default safe script set
nmap --script=vuln target                # vulnerability-focused scripts
nmap --script=smb-enum-shares target     # specific script
nmap --script "http-*" target            # wildcard script category
nmap --script-help=smb-enum-shares       # what a script does before you run it
```

Common categories: `auth`, `broadcast`, `default`, `discovery`, `dos`, `exploit`, `external`, `fuzzer`, `intrusive`, `malware`, `safe`, `version`, `vuln`.

## 8. Timing & Performance

```
nmap -T0 target   # paranoid, very slow, IDS evasion
nmap -T1 target   # sneaky
nmap -T2 target   # polite
nmap -T3 target   # normal (default)
nmap -T4 target   # aggressive, most common for lab/CTF use
nmap -T5 target   # insane, fastest, sacrifices accuracy
```

Custom control beyond the T0-T5 presets, useful when a preset is close but not quite right:

```
nmap --max-rate 10 target          # cap packets/sec sent, throttle below IDS/rate-limit thresholds
nmap --min-rate 100 target         # floor packets/sec, force faster sending when accuracy matters less
nmap --scan-delay 1s target        # fixed delay between probes to a single host
nmap --max-scan-delay 10ms target  # cap adaptive delay so it doesn't grow too slow
```

`--max-rate` is the practical go-to for staying under a firewall/IDS's connection-rate alerting threshold. Most detection rules trigger on *how many* ports get touched *how fast*, not on that a scan happened at all. Slower and quieter beats a preset timing template if the target is actively monitored.

## 9. Firewall / IDS Evasion

```
nmap -f target                     # fragment packets
nmap -mtu 24 target                # custom fragment size (multiple of 8)
nmap -D RND:10 target              # decoy scan, 10 random decoy IPs
nmap -D decoy1,decoy2,ME target    # decoy scan with explicit IPs, ME = your real position
nmap -sI zombie_host target        # idle/zombie scan, scan via a third host, fully blind
nmap --source-port 53 target       # spoof source port (bypass port-based filter rules)
nmap --spoof-mac 0 target          # random MAC spoof (local network only)
nmap --data-length 25 target       # pad packets to avoid signature-based length matching
nmap --max-rate 10 -T2 target      # combine rate cap + slow timing template for a genuinely quiet scan
```

Idle scan requires a zombie host with sequential/predictable IPID and no traffic of its own. Increasingly rare on modern networks, but useful conceptually for understanding blind scanning.

Realistic evasion is rarely one flag. It's usually `--max-rate` (or a slow `-T` template) stacked with fragmentation and/or decoys, tuned to whatever the target's detection is actually keying on.

## 10. Output Formats

```
nmap -oN scan.txt target        # normal output
nmap -oX scan.xml target        # XML, parseable, feeds into other tools
nmap -oG scan.gnmap target      # grepable
nmap -oA scan_basename target   # all three formats at once
```

## 11. Go-To Recipe Combos

```
# Fast initial sweep
nmap -sS -T4 -F target

# Full TCP port sweep, minimal noise
nmap -p- -T4 target

# Full detail pass once ports are known
nmap -sC -sV -O -p<ports> target -oA full_scan

# UDP top ports (UDP is slow, narrow scope)
nmap -sU --top-ports 50 -T4 target

# Quick vuln check pass
nmap -sV --script=vuln target
```

Typical workflow: fast discovery sweep, then full port range to find everything open, then a targeted `-sC -sV` detail pass on discovered ports, then NSE vuln/enum scripts on interesting services.

## 12. Field Notes

Practical lessons from actual enumeration work, not just syntax.

**SMB share enumeration: blank paths.** `smb-enum-shares` and `enum4linux` often return a blank share path when hitting Samba on Linux targets. The reliable fallback is:

```
rpcclient -U "" -N target
netshareenumall
```

This returns paths in Windows notation (e.g. `C:\home\user\`) even against a Linux Samba box. Translate literally: drop the `C:`, flip `\` to `/`, drop the trailing slash. The `path:` value returned is already the *full* directory. Don't append the share name on top of it, that produces a wrong path.

**General reminder:** when a standard enumeration script comes back empty or malformed, don't assume the service is misconfigured. Try a lower-level tool (`rpcclient`, raw `nc`, manual protocol interaction) before moving on. Automated scripts fail silently more often than they error loudly.
