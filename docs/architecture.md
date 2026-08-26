# CyphornIPS Systems & Developer Architecture

This document provides a technical deep-dive into the internal architecture, memory layout, IPC protocols, and lifecycle management of **CyphornIPS**.

---

## 1. High-Level Subsystems

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                             NETWORK INTERFACES                              │
│                    WAN (enp0s3)  <======>  LAN (enp0s8)                     │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ eBPF XDP / AF_PACKET Ring Buffer
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            STATEFUL DECODER                                 │
│  IPv4 / IPv6  ──►  TCP Reassembly  ──►  UDP / ICMP  ──►  Flow Tracker       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Sticky DPI Buffers & Extracted Files
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DETECTION PIPELINE                               │
│  ┌───────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │
│  │ Aho-Corasick Matcher  │  │ File Intelligence    │  │ Domain Policy    │  │
│  │ (Rules & Signatures)  │  │ (Exact Hash Matcher) │  │ (DNS Whitelist)  │  │
│  └───────────┬───────────┘  └──────────┬───────────┘  └────────┬─────────┘  │
└──────────────┼─────────────────────────┼───────────────────────┼────────────┘
               └─────────────────────────┼───────────────────────┘
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ACTION AGGREGATION                                 │
│                   Precedence: PASS  >  DROP  >  ALERT                       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Verdict + UI Security Events
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ENFORCEMENT & UI                                 │
│  ┌───────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │
│  │ eBPF Map Sync         │  │ Ncurses Dashboard    │  │ /var/log/alerts  │  │
│  │ (Hardware Drop Map)   │  │ (Real-Time Screens)  │  │ (JSON / Fast Log)│  │
│  └───────────────────────┘  └──────────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       ▲
                                       │ Unix Domain Socket IPC (/run/control.sock)
┌──────────────────────────────────────┴──────────────────────────────────────┐
│                  UPDATE & CONTROL SUBSYSTEM (cyphornctl)                    │
│   Provider Adapter  ──►  SHA-256 Gate  ──►  Pre-Validation  ──►  Hot Reload │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Generation Synchronization Model

To guarantee zero-downtime hot reload and prevent race conditions across threads without holding coarse mutex locks during packet processing, CyphornIPS implements a three-tier atomic generation model:

1. **`rules_generation`**: Increments each time a new `RuleStore` is loaded into userspace.
2. **`detection_generation`**: Increments when the Aho-Corasick state machine and fast-pattern tables are committed to the detection pipeline.
3. **`enforcement_generation`**: Increments when eBPF kernel maps and hardware-offloaded blocking tables are synchronized.

```c
typedef struct EngineState
{
    /* Generation tracking */
    uint64_t rules_generation;
    uint64_t detection_generation;
    uint64_t enforcement_generation;

    /* Active Subsystems */
    CyphornRuleStore rules;
    CyphornFileIntelligence file_intelligence;
    CyphornAlertStore alerts;
    ConnectionTable connections;
    CyphornControlServer control;
    EngineConfig config;
    ...
} EngineState;
```

---

## 3. Control IPC Protocol

Communication between `cyphornctl` and `cyphorn-engine` occurs over a local Unix domain stream socket:
- **Primary Path**: `/run/cyphornips/control.sock`
- **Fallback Path**: `/tmp/cyphornips_control.sock`

### Protocol Flow:
1. `cyphornctl` connects and transmits a single newline-terminated command string:
   `reload rules\n`, `reload intelligence\n`, `reload all\n`, `status\n`, `status json\n`, or `ping\n`.
2. The engine non-blocking poll loop (`cyphorn_control_server_poll()`) accepts the connection, parses the command, dispatches the internal reload/telemetry routines, and streams back the result.
3. Socket closes cleanly; zero persistent thread overhead.

---

## 4. File Structure & Code Organization

| Subsystem | Source Files | Header Files |
|---|---|---|
| **Rule Engine & Fingerprinting** | `src/rule.c`, `src/rule_parser.c`, `src/rule_loader.c`, `src/rule_ownership.c` | `include/rule.h`, `include/rule_loader.h`, `include/rule_ownership.h` |
| **Action Aggregation** | `src/action.c` | `include/action.h` |
| **Update Engine & Providers** | `src/update.c`, `src/http_fetch.c` | `include/update.h` |
| **Control IPC & CLI** | `src/control.c`, `src/cyphornctl.c` | `include/control.h` |
| **DPI & File Intelligence** | `src/file_intelligence.c`, `src/file_inspection.c`, `src/file_extraction.c` | `include/file_intelligence.h`, `include/file_inspection.h` |
| **Protocol Dissectors** | `src/http.c`, `src/tls.c`, `src/dns.c`, `src/ftp.c`, `src/icmp.c`, `src/tcp.c` | `include/http.h`, `include/tls.h`, `include/dns.h`, `include/ftp.h` |
| **UI Dashboard & Telemetry** | `src/ui.c` | `include/ui.h` |
