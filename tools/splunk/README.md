# Splunk Reference

Practical notes from standing up Splunk Enterprise for SIEM/log analysis lab work — setup, core search syntax, and the real troubleshooting lessons that came out of it.

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Installation](#2-installation)
3. [Getting Data In](#3-getting-data-in)
4. [SPL Basics](#4-spl-basics)
5. [Time Range Gotcha](#5-time-range-gotcha)
6. [Go-To Searches](#6-go-to-searches)
7. [Field Notes](#7-field-notes)

---

## 1. Core Concepts

| Term | Meaning |
|---|---|
| **Index** | Where ingested data is stored — think of it as a dedicated bucket/database for a data source |
| **Sourcetype** | Format label Splunk uses to parse an event (e.g. `csv`, `syslog`, `json`) |
| **Event** | One record — a line (or parsed unit) of ingested data with a timestamp |
| **Field** | A key/value extracted from an event, either automatically or via a search-time extraction |
| **SPL** | Search Processing Language — Splunk's query language for searches, `stats`, `eval`, etc. |

## 2. Installation

```
# Linux tarball install
sudo ./splunk start --accept-license --run-as-root

# Docker (fastest path for a disposable lab instance)
docker run -d -p 8000:8000 \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='yourpass' \
  splunk/splunk
```

**The "running as root" warning on startup is a deprecation notice, not an error.** It's fine for lab/practice work. A dedicated non-root `splunk` service account is a production best-practice, not a functional requirement.

**VM sizing:** Splunk is memory-hungry. 4GB RAM is undersized and will OOM-crash the instance under real search load — 8GB RAM / 4 vCPUs is a safer minimum for a lab VM.

## 3. Getting Data In

```
splunk add oneshot ~/sample.csv -sourcetype csv -index sample_index
```

`oneshot` is the fastest way to ingest a single file for lab/testing purposes without setting up a full forwarder or monitor input.

## 4. SPL Basics

```
index=sample_index                                  # pull everything from an index
index=sample_index authentication_status=failed      # filter on a field value
... | stats count by username                        # aggregate — count events per value
... | table timestamp, username, source_ip            # shape output to specific columns
... | timechart count by authentication_status        # time-bucketed aggregation, good for graphs
... | eval status_flag=if(authentication_status="failed", "ALERT", "ok")   # derived field
... | rex field=_raw "user=(?<user>\w+)"               # regex field extraction at search time
```

## 5. Time Range Gotcha

**The single most common "why is there no data" cause: the default time-range picker is "Last 24 hours."** If your sample/lab data is timestamped in the past (a downloaded dataset, an older PCAP conversion, anything not generated live), it will not appear until you switch the picker to **All time**. Check this before touching indexes, sourcetypes, or ingestion — it's the fastest thing to rule out and the most common reason data "isn't there."

## 6. Go-To Searches

```
# Failed logins by user, ranked
index=sample_index authentication_status=failed | stats count by username | sort -count

# Spike detection over time
index=sample_index | timechart span=1h count by authentication_status

# Distinct source IPs hitting a single account (possible brute force / credential stuffing)
index=sample_index authentication_status=failed | stats dc(source_ip) as unique_ips by username | where unique_ips > 3
```

## 7. Field Notes

Real lessons from setup and lab troubleshooting — the kind of debugging that's the actual job, not just the syntax.

**"No data anywhere" — check the source file before chasing Splunk.** Hit a case where `index=*` returned nothing, oneshot ingest showed nothing, and the upload preview was blank. After several rounds of checking sourcetype, index, and time range, the actual root cause was that the source CSV file on disk was 0 bytes (silently truncated). **Lesson: when Splunk shows no data at all, run `ls -l` on the source file first and confirm its size is non-zero before troubleshooting anything Splunk-side.** A corrupt or empty source file reproduces every symptom of a Splunk misconfiguration.

**Disk space halts searches silently.** Splunk enforces a minimum free disk space (default 5000MB) on the search dispatch directory and will stop running searches if the volume drops below it — with an error that has nothing obviously to do with disk space at first glance. Quick check: `df -h` on the volume backing `$SPLUNK_HOME/var/run/splunk/dispatch`. Band-aid: `splunk set minfreemb 500` + restart. Real fix: give the box more disk.

**Resizing a VM disk is a two-step operation, not one.** Growing the virtual disk at the hypervisor level (e.g. via `qemu-img` or your VM manager's disk resize) only grows the *outer container*. The guest OS's partition and filesystem stay their original size until you grow them from inside the guest:

```
# Inside the guest, after the hypervisor-level resize:
sudo growpart /dev/vda 2      # grow the partition to fill the new disk space
sudo resize2fs /dev/vda2      # grow the filesystem to fill the partition
df -h /                       # confirm the new size actually shows up
```

For an LVM layout instead of a plain partition: `pvresize` → `lvextend` → `resize2fs`. Forgetting the in-guest half of this is a common trap — the disk "looks" resized at the hypervisor but the OS still reports the old size and errors persist.
