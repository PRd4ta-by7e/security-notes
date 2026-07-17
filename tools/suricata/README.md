# Suricata Reference

Practical reference for Suricata as a network IDS/IPS — running modes, rule syntax, and log analysis.

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Running Modes](#2-running-modes)
3. [Rule Syntax](#3-rule-syntax)
4. [Log Output (eve.json)](#4-log-output-evejson)
5. [Useful Commands](#5-useful-commands)
6. [Go-To Workflow](#6-go-to-workflow)

---

## 1. Core Concepts

| Term | Meaning |
|---|---|
| **IDS mode** | Passive — inspects traffic and alerts, does not block |
| **IPS mode** | Active — inline, can drop/reject matching traffic |
| **Rule** | A signature defining what traffic pattern triggers an alert/action |
| **eve.json** | Suricata's structured JSON event log — the primary output for analysis and SIEM ingestion |
| **Ruleset** | A collection of rules, usually from a maintained source (ET Open, Suricata's own, custom) |

## 2. Running Modes

```
suricata -c /etc/suricata/suricata.yaml -i eth0        # live interface, IDS mode
suricata -c /etc/suricata/suricata.yaml -r capture.pcap # offline analysis of a pcap file
suricata -T -c /etc/suricata/suricata.yaml              # test config validity without running
```

IPS (inline) mode requires NFQUEUE or AF_PACKET inline configuration — a deliberate deployment choice, not the default.

## 3. Rule Syntax

Basic anatomy of a rule:

```
alert tcp any any -> $HOME_NET 22 (msg:"Possible SSH brute force"; flow:to_server; threshold:type threshold, track by_src, count 5, seconds 60; sid:1000001; rev:1;)
```

Breakdown:
- `alert` — action (`alert`, `drop`, `reject`, `pass`)
- `tcp any any -> $HOME_NET 22` — protocol, source IP/port → destination IP/port
- `msg:"..."` — human-readable alert description
- `flow:to_server` — directionality constraint
- `threshold:...` — rate-based condition (e.g. 5 hits in 60s = likely brute force, not one-off)
- `sid:` — unique rule ID (custom rules should use IDs outside the range reserved for public rulesets, typically 1000000+)
- `rev:` — rule revision number

Common keywords worth knowing: `content:"..."` (payload match), `pcre:"..."` (regex match), `dsize:` (packet size), `flags:` (TCP flags), `classtype:` (alert category).

## 4. Log Output (eve.json)

Suricata's `eve.json` is the primary structured log — every alert, flow, DNS query, TLS handshake, and HTTP transaction becomes a JSON event. Filtering it with `jq` is the fastest way to triage without a full SIEM in front of it:

```
# All alerts, most recent last
jq 'select(.event_type=="alert")' eve.json

# Alert messages + source/dest IPs only, compact view
jq -c 'select(.event_type=="alert") | {msg: .alert.signature, src: .src_ip, dest: .dest_ip}' eve.json

# Count alerts by signature, highest first
jq -r 'select(.event_type=="alert") | .alert.signature' eve.json | sort | uniq -c | sort -rn
```

## 5. Useful Commands

```
suricata-update                          # fetch/update rulesets (ET Open by default)
suricata-update list-sources             # see available ruleset sources
suricata-update enable-source <source>   # add a ruleset source
suricatasc -c "reload-rules"             # hot-reload rules on a running instance via unix socket
```

## 6. Go-To Workflow

Typical triage pass on a capture or live segment:

1. Validate config: `suricata -T -c suricata.yaml`
2. Run against target traffic (live interface or pcap)
3. Tail `eve.json` for alerts as they land, or batch-analyze after a pcap run
4. Pivot from a `sid` of interest back into the matching rule to understand exactly what triggered it — don't treat an alert message as self-explanatory without checking the rule logic behind it
5. Cross-reference `flow` and `http`/`dns`/`tls` events around the alert timestamp for context — an isolated alert rarely tells the whole story on its own
