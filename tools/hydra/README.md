# Hydra Reference

Practical reference for Hydra: online brute-force/password-guessing against live login services.

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [Basic Syntax](#2-basic-syntax)
3. [Common Protocols](#3-common-protocols)
4. [Field Notes](#4-field-notes)

---

## 1. Core Concept

Unlike Hashcat (offline, against a captured hash), Hydra attacks a live service directly: every attempt is a real network authentication request against the target. That makes it slower, noisier, and risk of account lockout is real, unlike offline cracking.

## 2. Basic Syntax

```
hydra -l admin -p Password123 target service          # single user, single password
hydra -l admin -P rockyou.txt target service           # single user, password list
hydra -L users.txt -P rockyou.txt target service       # user list + password list (full spray)
```

`-l`/`-L` = login name(s), lowercase single / uppercase list. Same pattern for `-p`/`-P`.

## 3. Common Protocols

```
hydra -l admin -P rockyou.txt ssh://10.10.10.10
hydra -L users.txt -P rockyou.txt rdp://10.10.10.10
hydra -l admin -P rockyou.txt 10.10.10.10 smb

# HTTP POST login form, the trickiest syntax, has to match the form's exact fields and failure text
hydra -l admin -P rockyou.txt target.com http-post-form \
  "/login:username=^USER^&password=^PASS^:F=incorrect"
```

For the HTTP form variant: `^USER^`/`^PASS^` mark where Hydra substitutes each attempt, and `F=<string>` tells Hydra what text in the response means "failed login". Get that string wrong and every attempt reads as a false success or false failure.

## 4. Field Notes

**Prefer targeted spraying over full brute force whenever you can.** A full `users.txt` × `rockyou.txt` combination against a live service is slow and will trigger account lockout policies fast. If the goal is finding *any* valid credential (common in an internal assessment), a **password spray**, trying one or two likely passwords (season+year, company name variants) against every known username, is far safer and often more effective than brute-forcing one account hard.

**For AD/SMB targets specifically, CrackMapExec is usually the better tool than Hydra.** It validates credentials the same way but also tells you immediately whether a hit means local admin (`Pwn3d!`), which Hydra has no concept of. Reach for Hydra mainly for non-SMB services (SSH, RDP, web login forms) where CrackMapExec doesn't apply.
