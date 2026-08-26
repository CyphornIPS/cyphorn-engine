# CyphornIPS Telemetry, Rule Statuses & UI Metrics

This document provides a comprehensive reference for all telemetry fields, status tags, badges, and accounting metrics in **CyphornIPS**. It covers both the interactive Ncurses Dashboard and the CLI/JSON runtime telemetry.

---

## 1. Rule Status Badges & Abbreviations

In the CyphornIPS Dashboard (specifically on the `[4] RULES` tab and in **Rule Details**), each rule is classified into one of the following authoritative operational statuses:

| Status Badge | Abbreviation | Meaning | When It Appears | Effect on Traffic |
|---|:---:|---|---|---|
| `[IDLE]` | `IDLE` | Never matched any packet since engine start or last reload. | `matched_packets == 0` | Packet unaffected by this rule. |
| `[MATCHED]` | `MATCHED` | The rule's signature criteria (IPs, ports, content, DPI buffers) matched at least one packet. | `matched_packets > 0` | Criteria satisfied; verdict determined by action precedence. |
| `[EFFECTIVE]` | `EFF` | The rule matched AND won the action aggregation precedence, dictating the final packet verdict. | `effective_packets > 0` | Dictates packet verdict (`ALLOW`, `DROP`, or `ALERT`). |
| `[OVERRIDDEN]` | `OVRD` | The rule matched the packet, but was overridden by another matching rule with higher action precedence. | `overridden_packets > 0` and `effective_packets == 0` | Rule had zero effect on the final verdict. |
| `[DROPPED]` | `DROP` | The rule is an effective `DROP` or `REJECT` rule that actively caused packets to be dropped inline. | `dropped_packets > 0` | Packet was discarded/blocked at the network layer. |
| `[ALERTED]` | `ALERT` | The rule is an effective `ALERT` rule that generated security alert events. | `alert_events > 0` | Packet forwarded, security event logged. |

---

## 2. Rule Matching vs. Effective Enforcement

A common point of confusion in network security monitoring is the difference between a **rule match** and **effective packet enforcement**.

### The Two-Stage Evaluation Pipeline

$$\text{Packet Arrival} \xrightarrow{\text{Stage 1: Signatures}} \text{Matching Rules Set} \xrightarrow{\text{Stage 2: Action Aggregation}} \text{Single Effective Verdict}$$

1. **Stage 1 (Signature Matching)**: CyphornIPS evaluates all active rules against the packet. Every rule whose protocol, IP, port, and pattern conditions are met increments its `matched_packets` counter.
2. **Stage 2 (Action Aggregation)**: The engine compares the actions of all matched rules using strict action precedence:
   $$\mathbf{PASS} > \mathbf{DROP} > \mathbf{ALERT}$$
   - The winning rule is marked **`EFFECTIVE`** and increments `effective_packets`.
   - All subordinate rules in the match set are marked **`OVERRIDDEN (OVRD)`** and increment `overridden_packets`.

### Telemetry Accounting Matrix Examples

#### Scenario A: PASS Rule Overrides DROP Rule (Whitelist Override)
- **Rule A (SID 1000003)**: `drop icmp 192.168.10.0/22 any -> any any (msg:"DROP ICMP"; sid:1000003; rev:1;)`
- **Rule B (SID 9999001)**: `pass icmp any any -> any any (msg:"ALLOW ICMP"; sid:9999001; rev:1;)`
- **Traffic**: Client `192.168.10.30` sends 41 ICMP Echo Requests.

| Rule SID | Action | MATCHED | EFFECTIVE | OVERRIDDEN (OVRD) | DROPPED | ALERTS | Final Verdict |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **9999001** | `PASS` | **41 (YES)** | **41 (YES)** | **0 (NO)** | 0 (NO) | 0 | **ALLOW (Forwarded)** |
| **1000003** | `DROP` | **41 (YES)** | **0 (NO)** | **41 (YES)** | **0 (NO)** | **0** | *Overridden by PASS* |

> [!IMPORTANT]
> Because SID 1000003 was overridden by PASS, **zero packets were dropped** and **zero false alerts** were written to `/var/log/cyphornips/alerts.log`.

#### Scenario B: DROP Rule Alone Matches
- **Rule A (SID 1000002)**: `drop tcp any any -> any 23 (msg:"BLOCK TELNET"; sid:1000002; rev:1;)`
- **Traffic**: 10 Telnet connection attempts.

| Rule SID | Action | MATCHED | EFFECTIVE | OVERRIDDEN | DROPPED | ALERTS | Final Verdict |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **1000002** | `DROP` | **10 (YES)** | **10 (YES)** | **0 (NO)** | **10 (YES)** | **10 (YES)** | **DROP (Blocked)** |

#### Scenario C: ALERT Rule Alone Matches
- **Rule A (SID 2000005)**: `alert tcp any any -> any 80 (msg:"HTTP TRAFFIC"; sid:2000005; rev:1;)`
- **Traffic**: 5 HTTP GET requests.

| Rule SID | Action | MATCHED | EFFECTIVE | OVERRIDDEN | DROPPED | ALERTS | Final Verdict |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **2000005** | `ALERT` | **5 (YES)** | **5 (YES)** | **0 (NO)** | **0 (NO)** | **5 (YES)** | **ALLOW (Logged)** |

---

## 3. Interactive Ncurses Dashboard Structure

When launched interactively (`cyphorn-engine -w wan0 -l lan0`), CyphornIPS displays a high-performance terminal UI divided into the following sections:

### 3.1 Global Header (Top 5 Rows)
```
╔════════════════════════════════════════════════════════════════════════════════════════════════╗
║ CyphornIPS v1.0.0 | WAN: eth0 [UP] <-> LAN: eth1 [UP] | PPS: 14,230 | BPS: 112.4 Mbps         ║
║ [1] LIVE      [2] FLOWS      [3] ALERTS      [4] RULES                                         ║
╚════════════════════════════════════════════════════════════════════════════════════════════════╝
```
- **WAN / LAN Status**: Interface link state and forwarding mode.
- **PPS / BPS**: Real-time throughput calculated on 1-second rolling averages.
- **System Telemetry**: System RAM %, Engine RSS KB, Total CPU %, Engine Proc CPU %.

### 3.2 Navigation Screens
- **`[1] LIVE`**: Split-view showing live packet inspection rates, memory tables, and a real-time event stream.
- **`[2] FLOWS`**: Stateful connection tracking table displaying 5-tuple, protocol, state (e.g. `ESTABLISHED`, `FIN_WAIT`), byte counters, and application protocol tag.
- **`[3] ALERTS`**: Security alert log viewer with timestamp, SID, severity, action, and signature message.
- **`[4] RULES`**: Master rule viewer with interactive search, match filtering (`[M]`), and detailed rule inspection.

---

## 4. Rule Inspection Telemetry Fields

Pressing `ENTER` or `TAB` on any rule in the `[4] RULES` screen opens the **Rule Details Inspector**, displaying exact metrics:

```
────────────────────────── RULE METRICS & ACCOUNTING ──────────────────────────
  SID             : 1000003
  Revision        : 1
  Ownership       : LOCAL (custom.rules)
  Fingerprint     : 4a6c7b27a386ed575c595cbb067158f6daaa05ce70fc941d2e4af9d42848d199
  Action          : DROP
  Protocol        : ICMP
  Source          : 192.168.10.0/22:any
  Destination     : any:any

  Status          : OVERRIDDEN
  Matches         : 41
  Effective       : 0
  Overridden      : 41
  Alerts          : 0
  Dropped         : 0
  Last Match      : 14:22:01.814

────────────────────────── WHY THIS RULE MATCHED ──────────────────────────────
  Buffer          : ICMP_HEADER
  Operator        : itype
  Pattern         : 8 (Echo Request)
  Matched Value   : 8
  Fast Pattern    : HIT
  Full Evaluation : MATCH
```

---

## 5. Engine Runtime Telemetry via CLI

Running `cyphornctl status` or `cyphornctl status json` queries the engine via the Unix control socket and outputs operational metrics.

### Plain Text (`cyphornctl status`)
```text
================================================================================
                    CyphornIPS Status & Runtime Telemetry                       
================================================================================
  Engine State       : RUNNING
  Process ID (PID)   : 24634
  Uptime             : 3h 24m 12s
  Control Socket     : /run/cyphornips/control.sock

------------------------------ Rules Subsystem --------------------------------
  Loaded Rules       : 3204
  Rules Directory    : /etc/cyphornips/rules
  Rules Generation   : 4
  Detection Gen      : 4
  Enforcement Gen    : 4
  Reload Count       : 3
  Last Reload Time   : 2026-08-26 00:45:12
  Last Reload Result : SUCCESS (3204 rules loaded, 0 errors)

------------------------ File Intelligence Subsystem --------------------------
  Status             : ENABLED
  Dataset File       : /etc/cyphornips/rules/cyphorn-file-intelligence.json
  Intelligence Gen   : 3
  Loaded Records     : 1500
  Reload Count       : 2
  Last Reload Time   : 2026-08-26 00:45:12
  Last Reload Result : SUCCESS (1500 threat records loaded)

------------------------- Installed Version Metadata --------------------------
  Rules Version      : 2026.08.25.1 (3204 rules)
  Intelligence Ver   : 2026.08.25.1 (1500 records)
  Update Channel     : stable
  Last Update Status : SUCCESS
================================================================================
```

### JSON Format (`cyphornctl status json` or `cyphornctl status --json`)
```json
{
  "engine_state": "RUNNING",
  "pid": 24634,
  "uptime_seconds": 12252,
  "rules": {
    "loaded_rules": 3204,
    "rules_directory": "/etc/cyphornips/rules",
    "generation": 4,
    "detection_generation": 4,
    "enforcement_generation": 4,
    "reload_count": 3,
    "last_reload_timestamp": 1756165512,
    "last_reload_success": true,
    "last_reload_message": "OK: Loaded 3204 rules (0 parse errors)"
  },
  "file_intelligence": {
    "enabled": true,
    "dataset_file": "/etc/cyphornips/rules/cyphorn-file-intelligence.json",
    "generation": 3,
    "loaded_records": 1500,
    "reload_count": 2,
    "last_reload_timestamp": 1756165512,
    "last_reload_success": true,
    "last_reload_message": "OK: Loaded 1500 threat intelligence records"
  },
  "installed_version": {
    "channel": "stable",
    "rules_version": "2026.08.25.1",
    "rules_count": 3204,
    "rules_sha256": "cbd01a33a50ba49d1fe0be6cb02cb996f41e8e195eec24818fa0cdf29dd294d6",
    "intel_version": "2026.08.25.1",
    "intel_record_count": 1500,
    "intel_sha256": "4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a",
    "last_update_status": "SUCCESS"
  }
}
```
