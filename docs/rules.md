# CyphornIPS Rule Engine & Ownership Architecture

This document details the rule syntax, supported inspection buffers, the **Managed vs. Local** storage architecture, and the **Canonical Rule Fingerprint** identity system in **CyphornIPS**.

---

## 1. Directory Structure: Managed vs. Local Rules

CyphornIPS strictly separates upstream/vendor-managed detection signatures from administrator-defined local policies on disk:

```text
/etc/cyphornips/rules/
├── managed/
│   └── rules.rules          # Downloaded & managed exclusively by CyphornIPS Update Engine
└── local/
    ├── custom.rules         # User-defined detection & blocking rules
    ├── whitelist.rules      # User-defined PASS override rules
    └── ...                  # Any additional local rule files (*.rules)
```

### Authoritative Boundaries:
- **`/etc/cyphornips/rules/managed/`**:
  - Contains candidate signatures downloaded from GitHub or generic HTTP update providers.
  - Managed **exclusively** by `cyphornctl update`.
  - Replaced atomically during successful updates or restored during rollbacks.
- **`/etc/cyphornips/rules/local/`**:
  - Reserved for local administrators.
  - **Protected Area**: The CyphornIPS update system is **strictly forbidden** from modifying, overwriting, or deleting any file within `local/`.
  - Survives all updates, rollbacks, and migrations untouched.

---

## 2. Canonical Rule Fingerprinting

### Why SID Alone Is Not an Identity
In traditional IDS/IPS systems, rule identity is tracked solely by the Snort Rule ID (`sid:`). However, this creates severe failure modes during updates:
- An administrator may write a local rule with `sid:1000003` to override an action (e.g. changing `alert` to `drop` or modifying IP ranges).
- If upstream releases a new rule package with `sid:1000003`, a SID-only updater would either delete or overwrite the user's custom rule.

### The Canonical Fingerprint Solution
CyphornIPS computes a deterministic 64-character SHA-256 fingerprint from the **canonical representation** of every parsed rule:

$$\text{Raw Rule String} \xrightarrow{\text{Parser}} \text{CyphornRule Struct} \xrightarrow{\text{Canonicalizer}} \text{Normalized String} \xrightarrow{\text{EVP\_sha256()}} \text{64-Hex Fingerprint}$$

```text
Raw Rule:
  drop  tcp  192.168.1.0/24  any  ->  any  80  ( msg:"BLOCK HTTP" ;  sid:1001 ; rev:1 ; )

Canonical Representation:
  act:2|proto:6|dir:0|src:192.168.1.0/24:any|dst:any:80|sid:1001|rev:1|msg:BLOCK HTTP|

SHA-256 Fingerprint:
  4a6c7b27a386ed575c595cbb067158f6daaa05ce70fc941d2e4af9d42848d199
```

### Key Properties:
1. **Whitespace & Formatting Invariance**: Cosmetic changes (spaces, tabs, capitalization of keywords, parameter order) do **not** change the fingerprint.
2. **SID Collision Distinction**: If an upstream rule and a local rule share the same SID but have different actions or patterns, they generate different fingerprints and coexist without collision.
3. **Reconciliation & Release Tracking**: Stored in `/var/lib/cyphornips/rules/ownership.json` to track provenance across releases.

```json
{
  "schema_version": 1,
  "managed_rules_version": "2026.08.25.1",
  "total_rules": 2,
  "rules": [
    {
      "sid": 1001,
      "rev": 1,
      "fingerprint": "4060b09b15e356566b43291f22589812b74169ae23d89d45f13c5a0b00960147",
      "ownership": "MANAGED",
      "managed_version": "2026.08.25.1",
      "source_file": "managed/rules.rules"
    },
    {
      "sid": 1001,
      "rev": 1,
      "fingerprint": "4a6c7b27a386ed575c595cbb067158f6daaa05ce70fc941d2e4af9d42848d199",
      "ownership": "LOCAL",
      "managed_version": "local",
      "source_file": "local/custom.rules"
    }
  ]
}
```

---

## 3. Supported Rule Syntax & Keyword Reference

CyphornIPS supports standard Snort/Suricata rule format:

```snort
<action> <protocol> <source_ip> <source_port> <direction> <dest_ip> <dest_port> (<options>)
```

### 3.1 Actions
- `pass`: Whitelist allow (overrides drop/alert).
- `drop`: Inline discard packet.
- `reject` / `rejectsrc` / `rejectdst` / `rejectboth`: Drop packet and send active TCP RST or ICMP unreachable.
- `alert`: Log security alert and forward packet.

### 3.2 Protocols
- `ip`, `ipv4`, `ipv6`, `tcp`, `udp`, `icmp`, `icmpv6`, `http`, `tls`, `dns`, `ftp`, `ftp-data`.

### 3.3 Direction
- `->` : Unidirectional from Source to Destination.
- `<>` : Bidirectional match.

### 3.4 Sticky DPI Buffers
- `http.uri`: Normalized HTTP request URI.
- `http.host`: HTTP Host header.
- `http.method`: HTTP Request method (GET, POST, etc.).
- `http.user_agent`: User-Agent header value.
- `http.header`: Complete normalized HTTP headers.
- `http.client_body`: HTTP client request payload.
- `http.stat_code`: HTTP response status code (e.g. 200, 404, 500).
- `tls.sni`: TLS ClientHello Server Name Indication (SNI).
- `dns.query`: Dissected DNS query hostname (UDP & TCP).
- `file.data`: Reassembled HTTP/FTP file payload (up to 16 MiB).

### 3.5 Content Modifiers
- `content:"<pattern>";`: String or hex pattern (`|41 42 43|`).
- `nocase;`: Case-insensitive pattern matching.
- `depth:<bytes>;` / `offset:<bytes>;`: Bounded search offsets.
- `distance:<bytes>;` / `within:<bytes>;`: Relative match boundaries.
- `fast_pattern;`: Designates pattern for Aho-Corasick prefilter.

### 3.6 File & Checksum Options
- `filemd5:"<md5_hex>";`: Exact MD5 checksum match.
- `filesha1:"<sha1_hex>";`: Exact SHA-1 checksum match.
- `filesha256:"<sha256_hex>";`: Exact SHA-256 checksum match.
- `filemagic:"<mime>";`: MIME type detection.
- `fileext:"<ext>";`: Lowercased file extension.
- `filename:"<name>";`: Nominal extracted filename.
- `filestore;`: Save extracted artifact and JSON metadata to `/var/log/cyphornips/files/`.

---

## 4. Flat-to-Hierarchical Migration (`cyphornctl update migrate`)

If you are upgrading from an older flat rule directory layout (`/etc/cyphornips/rules/*.rules`):

```bash
cyphornctl update migrate
```

### What Migration Does:
1. Creates `managed/` and `local/` subdirectories.
2. Identifies vendor rules (`rules.rules`, `managed.rules`) and moves them to `managed/rules.rules`.
3. Identifies all other files (`custom.rules`, `local.rules`, `threats.rules`) and moves them to `local/`.
4. Computes canonical fingerprints for all rules and initializes `/var/lib/cyphornips/rules/ownership.json`.
5. Pre-validates the migrated layout in a staging sandbox and requests live engine hot-reload.
6. If any step fails, restores the original flat directory automatically with zero rule loss.
