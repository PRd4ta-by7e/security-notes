# Hashcat Reference

Practical reference for Hashcat — offline password/hash cracking. The tool itself is simple; getting good results is mostly about picking the right mode and attack strategy.

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [Identifying a Hash First](#2-identifying-a-hash-first)
3. [Common Hash Modes](#3-common-hash-modes)
4. [Attack Modes](#4-attack-modes)
5. [Field Notes](#5-field-notes)

---

## 1. Core Concept

```
hashcat -m <mode> -a <attack-mode> hashes.txt wordlist.txt
```

Everything hinges on `-m` (which algorithm the hash actually is) — the wrong mode silently produces zero cracks even against a weak, crackable password, with no error telling you why.

## 2. Identifying a Hash First

```
hashid hash.txt          # quick guess at hash type
hash-identifier          # interactive alternative
```

Format also gives it away often: NTLM hashes are a single 32-char hex string; NTLMv2 captures (from Responder) include the full challenge/response blob, not just a hash.

## 3. Common Hash Modes

| Hash type | Mode | Where it comes from |
|---|---|---|
| NTLM | `-m 1000` | Windows SAM/local password hashes |
| NTLMv2 | `-m 5600` | Responder/LLMNR captured network auth |
| Kerberoast (TGS) | `-m 13100` | `GetUserSPNs.py` output |
| AS-REP | `-m 18200` | `GetNPUsers.py` output |
| WPA/WPA2 | `-m 22000` | Captured wireless handshakes |
| MD5 | `-m 0` | Legacy/web app password hashes |
| SHA-256 | `-m 1400` | Modern web app password hashes |
| bcrypt | `-m 3200` | Modern web app password hashes (salted, slow — much lower cracking speed) |

## 4. Attack Modes

```
hashcat -m 1000 -a 0 hashes.txt rockyou.txt                          # -a 0: straight dictionary attack
hashcat -m 1000 -a 0 hashes.txt rockyou.txt -r rules/best64.rule     # dictionary + mutation rules (case, leet-speak, appended digits)
hashcat -m 1000 -a 3 hashes.txt ?u?l?l?l?l?l?d?d                     # -a 3: mask/brute-force attack against a specific pattern
hashcat -m 1000 -a 6 hashes.txt rockyou.txt ?d?d                     # -a 6: wordlist + appended mask (e.g. word + 2 digits)
```

## 5. Field Notes

**Rules beat raw brute force for real-world passwords.** People append a digit or a year, capitalize the first letter, swap `a`→`@`. A dictionary attack with a solid rule file (`best64.rule`, `rockyou-30000.rule`) cracks far more real passwords per hour than a pure mask brute-force attempt at the same length, because it's testing patterns humans actually use instead of every possible combination blindly.

**Check `--show` before assuming a run failed.** Hashcat caches already-cracked hashes — if you rerun the same hash file, it may report "0 cracked" for the session while the answer is already sitting in the potfile from a previous run. `hashcat -m <mode> hashes.txt --show` surfaces anything already cracked without re-running the attack.
