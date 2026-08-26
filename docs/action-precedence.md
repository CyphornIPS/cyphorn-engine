# Rule Action Precedence & Conflict Resolution

This document explains the action precedence rules, conflict resolution mechanisms, and aggregation semantics implemented in the **CyphornIPS** detection engine.

---

## 1. Authoritative Action Precedence Order

When a network packet matches multiple detection rules simultaneously, CyphornIPS evaluates all matching rules and determines the single winning packet verdict using strict action precedence:

$$\mathbf{PASS} > \mathbf{DROP} \text{ (including } \mathbf{REJECT}\text{)} > \mathbf{ALERT}$$

### Priority Hierarchy in Code (`src/action.c`):
```c
enum
{
    CYPHORN_ACTION_PRIORITY_ALERT = 1,
    CYPHORN_ACTION_PRIORITY_DROP  = 2,
    CYPHORN_ACTION_PRIORITY_PASS  = 3
};
```

---

## 2. Action Reference & Semantics

### 2.1 `PASS` (Priority: 3 - Highest)
- **Description**: Explicit whitelist allow action.
- **Packet Fate**: The packet is immediately forwarded through the network bridge without being dropped or alerted.
- **When It Becomes Effective**: When matched, `PASS` overrides all concurrent `DROP`, `REJECT`, and `ALERT` rules.
- **Telemetry Accounting**:
  - `PASS` rule: `matched_packets++`, `effective_packets++`.
  - Overridden `DROP`/`ALERT` rules: `matched_packets++`, `overridden_packets++`, `dropped_packets == 0`, `alert_events == 0`.
- **Example Rule**:
  ```snort
  pass icmp any any -> any any (itype:8; msg:"CYPHORN ALLOW PING TO MANAGEMENT"; sid:9999001; rev:1;)
  ```

### 2.2 `DROP` & `REJECT` (Priority: 2 - Medium)
- **Description**: Inline blocking action.
- **Packet Fate**: The packet is discarded by the eBPF kernel filter or userspace bridge and not forwarded.
- **Supported Action Variants**:
  - `drop`: Silently discards the packet.
  - `reject`: Drops the packet and sends an active TCP RST or ICMP Port Unreachable response.
  - `rejectsrc`: Sends TCP RST/ICMP error to the source host.
  - `rejectdst`: Sends TCP RST/ICMP error to the destination host.
  - `rejectboth`: Sends TCP RST to both communication endpoints.
- **When It Becomes Effective**: Effective whenever at least one `DROP`/`REJECT` rule matches and **no** matching `PASS` rule exists.
- **Telemetry Accounting**:
  - Winning `DROP` rule: `matched_packets++`, `effective_packets++`, `dropped_packets++`, `alert_events++`.
  - Subordinate `ALERT` rules: `matched_packets++`, `overridden_packets++`, `alert_events == 0`.
- **Example Rule**:
  ```snort
  drop tcp 192.168.10.0/24 any -> any 23 (msg:"BLOCK INSECURE TELNET"; sid:1000002; rev:1;)
  ```

### 2.3 `ALERT` (Priority: 1 - Lowest)
- **Description**: Non-blocking signature detection and security logging.
- **Packet Fate**: The packet is forwarded untouched to its destination. An alert record is written to `/var/log/cyphornips/alerts.log` and forwarded to the interactive Dashboard.
- **When It Becomes Effective**: Effective when matched, provided **no** matching `PASS` or `DROP` rule exists.
- **Telemetry Accounting**:
  - Winning `ALERT` rule: `matched_packets++`, `effective_packets++`, `alert_events++`, `dropped_packets == 0`.
- **Example Rule**:
  ```snort
  alert tcp any any -> any 80 (http.uri; content:"/admin"; msg:"HTTP ADMIN ACCESS ATTEMPT"; sid:2000005; rev:1;)
  ```

---

## 3. Conflict Resolution Scenarios

### Scenario 1: `PASS` vs. `DROP`
```snort
# Rule 1 (Upstream Managed)
drop icmp 192.168.10.0/22 any -> any any (msg:"UPSTREAM DROP ICMP"; sid:1000003; rev:1;)

# Rule 2 (Local Override)
pass icmp 192.168.10.30 any -> 8.8.8.8 any (itype:8; msg:"LOCAL WHITELIST PING"; sid:9999001; rev:1;)
```
- **Packet**: `192.168.10.30` pings `8.8.8.8`.
- **Evaluation**: Both rules match.
- **Outcome**:
  - **Verdict**: `ALLOW` (Packet is transmitted).
  - **Rule 9999001 (`PASS`)**: `[EFFECTIVE]` (`effective_packets = 1`, `overridden = 0`).
  - **Rule 1000003 (`DROP`)**: `[OVERRIDDEN]` (`effective_packets = 0`, `overridden = 1`, `dropped = 0`).
  - **Alerts**: Zero alerts logged.

### Scenario 2: `DROP` vs. `ALERT`
```snort
# Rule 1
alert tcp any any -> any 443 (tls.sni; content:"malicious-c2.com"; msg:"ALERT C2 SNI"; sid:3000010; rev:1;)

# Rule 2
drop tcp any any -> any 443 (tls.sni; content:"malicious-c2.com"; msg:"BLOCK C2 SNI"; sid:3000020; rev:1;)
```
- **Packet**: Client initiates TLS ClientHello with SNI `malicious-c2.com`.
- **Evaluation**: Both rules match.
- **Outcome**:
  - **Verdict**: `DROP` (Packet blocked).
  - **Rule 3000020 (`DROP`)**: `[EFFECTIVE]` (`effective = 1`, `dropped = 1`, alert emitted for SID 3000020).
  - **Rule 3000010 (`ALERT`)**: `[OVERRIDDEN]` (`effective = 0`, `overridden = 1`).

### Scenario 3: Multiple Matching `DROP` Rules (Tie Breaking)
If multiple `DROP` rules match the same packet:
- The first matching rule encountered at the highest priority level becomes the **Effective / Winning Rule** (`winning_sid = 3000001`).
- The remaining matching `DROP` rules increment `matched_packets` and `overridden_packets`.
- Exactly **one** drop action and alert event is generated per packet to prevent alert storming.

---

## 4. Rule Ownership vs. Action Precedence

> [!IMPORTANT]
> **Rule ownership does NOT determine action precedence.**

- `LOCAL` rules do **NOT** automatically override `MANAGED` rules.
- `MANAGED` rules do **NOT** automatically override `LOCAL` rules.

| Concept | Question Answered | Controlling Factor |
|---|---|---|
| **Rule Ownership** | *"Who created this rule and which subsystem is allowed to update/delete it?"* | File directory location (`/etc/cyphornips/rules/managed/` vs. `/etc/cyphornips/rules/local/`). |
| **Action Precedence** | *"Which matching rule determines the packet's forwarding verdict?"* | Rule action keyword (`PASS` > `DROP` > `ALERT`), evaluated identically regardless of rule origin. |

If a network administrator wishes to whitelist traffic that is otherwise blocked by an upstream `MANAGED` drop rule, the administrator creates a local `PASS` rule in `/etc/cyphornips/rules/local/custom.rules`. The local rule wins because its action is **`PASS`**, not because it is local.
