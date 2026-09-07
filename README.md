---
# 🛡️ CyphornIPS v1.5.0 
---
Developer Preview


# High-Performance Inline Network Intrusion Prevention System

[![Language: C17](https://img.shields.io/badge/Language-C17-00599C.svg)](https://en.wikipedia.org/wiki/C17_(C_standard_revision))
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux%20Kernel%205.8%2B-FCC624.svg?logo=linux&logoColor=black)](https://kernel.org)
[![eBPF / TC](https://img.shields.io/badge/Kernel-eBPF%20%2F%20TC%20Filter-red.svg)](https://ebpf.io/)
[![Zero-Downtime Hot Reload](https://img.shields.io/badge/Engine-Zero--Downtime%20Hot%20Reload-brightgreen.svg)]()
[![Status: Alpha](https://img.shields.io/badge/Status-Developer%20Preview%20v1.5.0-orange.svg)]()


**CyphornIPS** is a transparent, inline Intrusion Prevention System (IPS) and Network Security Monitoring engine for Linux gateways and edge firewalls. Written in C17 with native eBPF/TC kernel acceleration, it bridges two dedicated network interfaces and performs deep packet inspection, stateful TCP stream reassembly, in‑memory file extraction, cryptographic exact‑hash threat‑intel matching, domain/country policy enforcement, and atomic zero‑downtime hot reload — all inline, at wire speed.

This release brings **per‑client policy attribution across NAT**, an interactive **"why was this blocked?" explain engine**, **SMTP/email inspection**, **threat‑feed datasets**, **flowbits**, **JA3 TLS fingerprinting**, **IPv6 enforcement**, and a long list of correctness fixes uncovered while hardening all of the above. Nothing below has been trimmed — every subsystem currently shipping in the engine is documented.

> [!WARNING]
> **Developer Preview / Alpha Release.** CyphornIPS v1.5.0 is provided for evaluation, security research, lab validation, and community testing. It is **not** certified as a sole, unmonitored security perimeter for mission‑critical production environments. See [Disclaimer & Operational Notice](#️-disclaimer--operational-notice-إخلاء-المسؤولية) below before deploying inline blocking.

---

## 📦 Installation & Upgrade

Two ways to install: the prebuilt Debian/Ubuntu package (recommended), or building from source. **This is also how you upgrade an existing install** — nothing under `/etc/cyphornips/` is ever overwritten if it already exists.

### Debian / Ubuntu Package (x86_64 / amd64, recommended)

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.5.0/cyphornips_1.5.0_amd64.deb

# Preferred: apt resolves any new library dependencies automatically
sudo apt install ./cyphornips_1.5.0_amd64.deb

# Equivalent with plain dpkg
sudo dpkg -i cyphornips_1.5.0_amd64.deb
sudo apt-get install -f -y   # only needed if dpkg reports missing dependencies

cyphorn-engine --version
cyphornctl --version
```

The package installs the engine, `cyphornctl`, the systemd unit, the compiled eBPF TC object, and — starting with this release — the bundled **GeoIP/ASN databases** and ready‑to‑use `cyphornips.conf`, `domain_policy.conf`, `country-policy.conf`, and `categories.conf` templates under `/etc/cyphornips/`.

**Upgrading an existing install:** the exact same command. `dpkg`/`apt` treat a newer `.deb` with the same package name as an upgrade, not a fresh install — your live `cyphornips.conf`, `domain_policy.conf`, `country-policy.conf`, and everything under `rules/` are **never part of the package payload**, so they're left exactly as they are; only the binaries, the systemd unit, and the BPF object get replaced.

```bash
# Upgrading: same command, no extra steps.
sudo apt install ./cyphornips_1.5.0_amd64.deb
```

If the service was already enabled (`systemctl is-enabled cyphornips` says `enabled`), it **stays enabled** across the upgrade — the old binary is stopped just long enough to unpack the new one. It is **not restarted automatically**, so run this once you're ready to run the new version:

```bash
sudo systemctl restart cyphornips
```

### Where Everything Lives

| Path | Type | Purpose |
|---|---|---|
| `/usr/bin/cyphorn-engine` | Binary | The core detection and forwarding engine |
| `/usr/bin/cyphornctl` | Binary | CLI for status, reload, `explain`, and updates |
| `/usr/lib/cyphornips/bpf/cyphornips_tc.bpf.o` | Object | Compiled kernel TC eBPF classifier |
| `/lib/systemd/system/cyphornips.service` | Service | Systemd unit |
| `/etc/cyphornips/cyphornips.conf` | Config | Primary engine configuration |
| `/etc/cyphornips/domain_policy.conf` | Config | Domain categories and scoped policy rules |
| `/etc/cyphornips/country-policy.conf` | Config | Geo‑fencing rules |
| `/etc/cyphornips/rules/managed/` | Directory | Upstream rules, owned by `cyphornctl update` |
| `/etc/cyphornips/rules/local/` | Directory | Your own rules — never touched by updates |
| `/etc/cyphornips/rules/cyphorn-file-intelligence.json` | Dataset | Normalized File Intelligence hash dataset |
| `/etc/cyphornips/update-signing.pub` | Public key | Pinned Ed25519 key used to verify update manifests |
| `/etc/cyphornips/geoip/` | Data | Bundled GeoIP/ASN `.mmdb` databases |
| `/var/log/cyphornips/{cyphornips,rules,alerts}.log` | Logs | Operational, rule‑lifecycle, and security‑alert streams |
| `/var/lib/cyphornips/version.json` | State | Installed rule/intelligence versions and timestamps |
| `/var/lib/cyphornips/updates/{staging,backup}/` | Directory | Update staging area and rollback snapshots |
| `/run/cyphornips/control.sock` | Socket | Unix control socket for `cyphornctl` |

### Requirements

| Requirement | Specification | Why it's needed |
|---|---|---|
| Operating system | Linux kernel ≥ 5.8 (6.x+ recommended) | eBPF Traffic Control (`clsact`) hooks, BPF hash maps, Netfilter conntrack integration |
| Network interfaces | 2 dedicated adapters (WAN & LAN) | Physical NICs (1/10/25/40/100GbE) or virtual (virtio, veth, tap, bridge ports) |
| Privileges | Root, or bounded capabilities | `CAP_NET_ADMIN`, `CAP_NET_RAW`, `CAP_BPF`, `CAP_SYS_RESOURCE` |
| Memory | 512 MB minimum, 2 GB+ recommended | TCP reassembly buffers, connection tracking, GeoIP databases, signature sets |
| Storage | 1 GB free minimum | Logs, rotated archives, extracted files, threat‑intel feeds |

Interface names are **never** hard‑coded — supplied at runtime via CLI flags, environment variables, or the config file.

### Minimal Working Configuration

```ini
wan_interface = wan0
lan_interface = lan0

rules_dir           = /etc/cyphornips/rules
domain_policy_file  = /etc/cyphornips/domain_policy.conf
country_policy_file = /etc/cyphornips/country-policy.conf

log_level = INFO
```

```bash
sudo systemctl enable --now cyphornips
sudo systemctl status cyphornips
cyphornctl status
```

---

## ✨ What's New in v1.5.0

This cycle's work centers on one theme: **a verdict is only useful if you can trust exactly who it applied to, and explain why it fired.** Alongside that, the protocol and detection surface grew substantially.

- 🧭 **Per‑client policy scope, fixed across NAT.** The engine inspects every conversation twice — once on the LAN side with the real client, and again post‑NAT on the WAN side, where the source is the router's own address. That second view used to fall through to `ANY`‑scoped rules and get logged against the router, making a working per‑device block indistinguishable from a missing one. The client is now resolved through the conntrack cache; where it can't be resolved, the engine skips the decision instead of guessing.
- 🔎 **`cyphornctl explain <ip> <domain>` — "why was this device blocked?"** A new interactive diagnosis command that prints every policy scope configured for the matched category, whether each one covers the client, and which single rule decided — plus a ready‑to‑paste line to fix it. See [Domain Policy & "Why Was I Blocked?"](#-domain-policy--why-was-i-blocked) below.
- 🩺 **New diagnostic capture script:** [`scripts/diagnose_client_block.sh`](scripts/diagnose_client_block.sh) `<client-ip> [seconds]` — captures the eBPF policy map, packet traces on both interfaces, and the relevant log windows for one client into a single directory, so a block can be root‑caused from one place instead of four logs and a map dump.
- 📧 **SMTP / email inspection.** Nine new sticky buffers matching Suricata 8's set: `email.from`, `email.to`, `email.cc`, `email.subject`, `email.date`, `email.message_id`, `email.x_mailer`, and the multi‑instance `email.url` and `email.received`, driven by a new SMTP/MIME dissector (command dialogue, RFC 5322 headers with folding and RFC 2047 encoded‑words, then the body — decoded before URL extraction so a base64 part can't hide a link).
- 📚 **Datasets — one rule, a whole threat feed.** `dataset:isset,<name>,load <path>;` tests membership of the active sticky buffer's value against an external file of tens of thousands of IPs, domains, hashes, or strings in **O(1)**, instead of one rule per indicator. Binds to any sticky buffer (`tls.sni`, `ja3.hash`, `http.host`, `email.url`, …).
- 🔗 **Flowbits — stateful correlation across packets.** `flowbits:set/unset/toggle/isset/isnotset` attach a named boolean to a flow, so one rule can mark state (e.g. "admin login happened") and a later rule can require it before firing (e.g. "only alert on command injection *within that same session*").
- 🔏 **JA3 client TLS fingerprinting**, exposed as `ja3.hash` / `ja3.string` sticky buffers — plus a fix to the engine's own MD5 implementation, which referenced the wrong message word and silently produced an incorrect digest for every non‑empty input. Every JA3 hash the engine ever computed before this fix was wrong; it now matches the published reference vectors.
- 🔁 **Dataset feeds now actually hot‑reload.** Rules held a pointer to their loaded list, and reload never dropped the cached copy — editing a feed and reloading reported success while the engine kept matching stale data. Both reload paths now invalidate the cache alongside the rules that reference it.
- 📝 **On‑demand TLS fingerprint logging** (`tls_log_ja3 = true`): records `[TLS_FINGERPRINT]` once per flow with client, hash, SNI, and ALPN — off by default since it costs a line per handshake, useful for building a fingerprint baseline from real traffic.
- 🧾 **ALLOW decisions are now logged too.** Previously only blocks were recorded, so "this device is allowed" was indistinguishable from "the policy never ran." `ACTION=PASS` is now recorded once per flow for any domain that matched a category, gated by `domain_policy_log_allow` (default on).
- 🏷️ **Every alert now carries `CLIENT=`** — the device the decision was actually scoped to — cleared per packet so one flow's resolved client can never bleed into a later alert, including on the QUIC cached‑decision repeat‑alert path.
- 🌐 **IPv6 enforcement datapath.** The kernel classifier had no IPv6 policy map, so IPv6 traffic was inspected and alerted on but never actually blocked. v1.5.0 adds the `cyphorn_policy_v6` map, IPv6 parsing in the classifier, TTL honoring, garbage collection, and address‑family tracking for QUIC flows.
- 🎯 **Country policy now sees the real client, too.** The same pre‑NAT resolution fix applied to domain policy was needed for country policy: a rule scoped to a LAN subnet was silently falling through to a broader `ANY` rule (or missing entirely) on the post‑NAT view.
- 🧵 **Multi‑instance sticky buffer matching fixed.** Both the fast‑pattern prefilter and the payload‑matching chain used to resolve a buffer to its *first* instance only — a rule matching a later `tls.subjectaltname`, `email.url`, or `email.received` value was silently discarded. Both now walk every instance; chained `content` matches stay anchored to the instance that matched first.
- ✅ **Regression suite reliability.** Fixed a suite that read the operator's live `domain_policy.conf` instead of its own fixture, a ~2.4 MB stack allocation that crashed the reload path under test, a stale hardcoded object list that silently broke linking when new modules were added, and wired up newly‑added suites (`test_smtp_email`, `test_email_keywords`, `test_policy_scope_nat`, `test_ipv6_enforcement_and_cache`) into `tests/run_all_tests.sh` so they actually run on every change.

---

## 🏗️ Architecture & Data Pipeline

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                 CyphornIPS — Architecture & Data Pipeline                  │
│                                                                            │
│  Ingress Packet on WAN / LAN                                               │
│        │                                                                   │
│        ▼                                                                   │
│  [ eBPF TC Ingress Filter ]  — line-rate drop before userspace             │
│        │                                                                   │
│        ▼                                                                   │
│  [ L2-4 Decoders: Ethernet / IPv4 / IPv6 / TCP / UDP / ICMP ]              │
│        │                                                                   │
│        ▼                                                                   │
│  [ Stateful Conntrack Table & Flow Tracker ]                               │
│        │                                                                   │
│        ▼                                                                   │
│  [ Protocol DPI Dissectors ]                                               │
│        │                                                                   │
│        ├── HTTP ─────► HTTP Parser & Sticky Buffers                        │
│        ├── TLS/QUIC ──► SNI, X.509 Certs & JA3 Client Fingerprint          │
│        ├── DNS ──────► DNS Wire Parser (UDP/TCP 53)                        │
│        ├── FTP ──────► Control Channel & Data Reassembler                  │
│        └── SMTP (NEW)► email.* Buffers (from/to/subject/url/...)           │
│               │                                                            │
│               ▼                                                            │
│  [ File Extraction (16 MiB max) → MD5 / SHA-1 / SHA-256 / SHA3-384 ]       │
│               │                                                            │
│               ▼                                                            │
│  [ Detection Engine: Aho-Corasick + Fast Pattern                           │
│    + Datasets (NEW) + Flowbits (NEW) ]                                     │
│               │                                                            │
│               ▼                                                            │
│  [ File Intelligence (SID 9100000) & Signature Rule Evaluation ]           │
│               │                                                            │
│               ▼                                                            │
│  [ Domain / Country Policy — real client, NAT-resolved (FIXED) ]           │
│               │                                                            │
│               ▼                                                            │
│  [ Action Precedence:   PASS   >   DROP / REJECT   >   ALERT ]             │
│        │                   │                      │                        │
│        ▼                   ▼                      ▼                        │
│   Forward,             Drop inline,            Forward,                    │
│   override any         sync eBPF policy      record CLIENT=                │
│   drop/alert           map, ready for         attributed alert             │
│                        cyphornctl explain                                  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Complete Feature Set

### Core Packet Capture & Protocol Decoding
- **Inline eBPF packet filtering:** transparent bridge forwarding with line‑rate early drops (`TC_ACT_SHOT`) in the kernel TC ingress hook, before a packet ever reaches user‑space.
- **Stateful protocol dissection:** full IPv4/IPv6 decoders, bidirectional TCP flow reassembly with out‑of‑order and duplicate segment handling, UDP, and ICMP correlation.
- Non‑blocking raw packet sockets (`AF_PACKET`, `SOCK_RAW`, `htons(ETH_P_ALL)`, `O_NONBLOCK`).
- **Automatic interface offload management:** native in‑process `ioctl(SIOCETHTOOL)` disables GRO on WAN/LAN at startup (`disable_interface_offload`, default `true`) — zero subprocess overhead, dynamic GRO bit discovery across kernel versions, no reliance on the external `ethtool` binary.
- **Strict DSA / switch‑master isolation:** on embedded topologies with a Distributed Switch Architecture master MAC (e.g. `eth0` on MediaTek MT7530/MT7988 platforms), that interface is never opened, bound, queried, or modified — operations are strictly confined to the configured WAN/LAN endpoints.

### Application‑Layer Deep Packet Inspection
- **HTTP/1.0 & HTTP/1.1:** sticky buffers `http.uri`, `http.uri.raw`, `http.host`, `http.method`, `http.user_agent`, `http.header`, `http.content_type`, `http.cookie`, `http.client_body`, `http.server_body`, `http.stat_code`.
- **SMTP / email (NEW):** `email.from`, `email.to`, `email.cc`, `email.subject`, `email.date`, `email.message_id`, `email.x_mailer`, multi‑instance `email.url` and `email.received`.
- **DNS wire inspection:** UDP and TCP parser (`dns.query`) with pointer‑loop protection and case normalization.
- **FTP data stream tracking:** dynamic tracking of active/passive control channels (`PORT`, `PASV`, `EPRT`, `EPSV`) and `STOR`/`RETR` payload extraction (`ftp-data`, `ftpdata_command`).
- **TLS handshake dissection:** SNI (`tls.sni`), X.509 certificate decoding (`tls.cert_subject`, `tls.cert_issuer`, `tls.cert_serial`, `tls.cert_fingerprint`, `tls.subjectaltname`), validity flags (`tls_cert_expired`, `tls_cert_valid`), negotiated version (`tls.version`).
- **JA3 client fingerprinting (NEW):** `ja3.hash` (32‑hex MD5 digest) and `ja3.string` (raw pre‑hash string), with an on‑demand `[TLS_FINGERPRINT]` log line gated by `tls_log_ja3`.
- **QUIC (HTTP/3) inspection:** decodes UDP/443 QUIC Initial packets for v1, v2, and draft‑29; derives Initial keys via HKDF; decrypts the CRYPTO frame (AES‑128‑GCM); recovers the TLS 1.3 ClientHello SNI. On a policy block, installs a **client‑scoped** eBPF UDP drop entry that doesn't affect other devices behind the same NAT.

### Payload Matching, Datasets & Flowbits
- Multi‑pattern content matching with `nocase`, `offset`, `depth`, `distance`, `within`, `startswith`, `endswith`, `rawbytes`, `fast_pattern`.
- **Datasets (NEW):** `dataset:isset|isnotset,<name>,load <path>;` — O(1) membership test against an external feed of any size, bound to whichever sticky buffer preceded it. A feed that fails to load makes the rule fail to match, never match everything. Feeds are hot‑reloadable as of this release.
- **Flowbits (NEW):** `flowbits:set|unset|toggle|isset|isnotset,<name>;`, plus `flowbits:noalert;` for silent state‑tracking rules. Conditions (`isset`/`isnotset`) are evaluated before content; actions (`set`/`unset`/`toggle`) only commit once the rule has matched in full, so a failed match never leaves a stray state on the flow. Bounded at 16 named bits per flow.
- `byte_test` / `byte_extract` for numeric/endian checks on packet, stream, and extracted‑file buffers.
- **IP, CIDR & GeoIP rules:** subnet CIDRs, address exclusion (`!`), TTL/TOS matching, and `geoip:[src|dst|both|any],[!]<ISO-code>[,...]`.
- **TCP/UDP/ICMP rules:** port ranges, TCP flags (`flags:S`, with `+`/`*`/`!` modifiers), connection state (`flow:established`), ICMP types/codes (`itype:8`, `icode:0`), and rate limiting (`threshold`).

### File Extraction & Inspection Subsystem
- Bounded in‑memory stream reassembly for HTTP requests/responses and FTP data transfers (up to **16 MiB** per file, **32 MiB** global concurrent ceiling, **4,096** chunks per file cap).
- On‑the‑fly hashing: **MD5, SHA‑1, SHA‑256, SHA3‑384** computed in a single OpenSSL EVP pass.
- Magic‑byte classification: PDF, JPEG, PNG (with parsed IHDR dimensions), GIF87a/89a (with screen dimensions), and Windows/DOS PE executables — independent of filename.
- MIME type, lowercased extension (`fileext`), and filename tracking (`filename` / `file.name`).
- Raw payload search via `file.data` with `content`, `nocase`, `depth`, `offset`, `distance`, `within`, `byte_test`.
- Checksum blacklists: `filemd5`/`filesha1`/`filesha256` against a `qsort()`‑sorted, `bsearch()`‑queried in‑memory IOC list — **O(log N)**, path‑traversal‑safe, strict hex‑length validation, missing file safely disables the rule rather than crashing.
- **Archiving (`filestore;`)**: saves the full file under its SHA‑256 name plus a structured JSON sidecar to `/var/log/cyphornips/files/` (override with `$CYPHORNIPS_FILE_STORE_DIR`); works across IPv4/IPv6 on both HTTP and FTP‑data; empty/truncated files are never stored.
- `noalert;` for silent collection — still matches and archives, just suppresses the visible alert.

### Exact‑Hash Threat Intelligence Engine
- Ingests a normalized JSON dataset (`/etc/cyphornips/rules/cyphorn-file-intelligence.json`), checked in order `sha256_hash` → `sha3_384_hash` → `sha1_hash` → `md5_hash`; a match on any populated field triggers detection.
- Optional metadata: `file_name`, `file_type_mime`, and a `tags` array.
- Exact‑hash matching only — no fuzzy hashing, reputation scoring, or ML classification.
- Fixed alert identifier: **SID 9100000**, classification `file-intelligence`.
- Invalid JSON is rejected outright at load/reload time; the previously‑loaded dataset stays active so inspection is never interrupted.

### Domain Policy & "Why Was I Blocked?"
- Two directives: `domain <name> <category> [patterns...]` to classify, and `policy <category> [scope] <BLOCK|ALLOW|DROP|PASS> [priority]` to enforce — over both DNS queries and TLS/QUIC SNI.
- `DROP`/`DENY`/`REJECT` are accepted spellings of `BLOCK`; `PASS` is an accepted spelling of `ALLOW`.
- **Most‑specific‑rule‑wins** resolution: exact host beats CIDR subnet beats wildcard `ANY`/`*`; priority only breaks ties *within* the same specificity tier — a wildcard rule at priority 1000 still loses to an exact‑host rule at priority 0.
- **Per‑client attribution across NAT (FIXED this release):** the pre‑NAT LAN‑side client is resolved through the conntrack cache and is authoritative; the post‑NAT WAN‑side view of the same conversation no longer overrides it or gets misattributed to the router's own address.
- **`cyphornctl explain <client-ip> <domain>` (NEW):** prints the matched category, every configured scope for it, whether each covers the client, which one decided, and a ready‑to‑paste config line to change the outcome — without guessing from logs.
- `ACTION=PASS` logging for allow decisions (NEW, gated by `domain_policy_log_allow`, default on) and `CLIENT=` attribution on every alert (NEW), so "allowed" is never indistinguishable from "never evaluated."

### Country Policy (Geo‑Fencing)
- `<allow|alert|drop> <inbound|outbound> <ISO-3166-1-alpha-2> <CIDR|ANY>` — acts directly on IPv4/IPv6 headers and Netfilter conntrack state, so it also covers traffic that never resolves through DNS.
- **First‑match‑wins**, evaluated top to bottom.
- **Pre‑NAT client resolution (FIXED this release):** the same NAT‑correctness fix applied to domain policy now applies here — a rule scoped to a LAN subnet is evaluated against the real local endpoint, not the router's post‑NAT address.
- Backed by bundled MaxMind/DB‑IP GeoIP & ASN databases for country/ASN log enrichment on alerts, rules, and the `geoip:` keyword.

### Rule Engine & Canonical Fingerprinting
- Hierarchical rule layout: `/etc/cyphornips/rules/managed/` (upstream, replaced only by `cyphornctl update`) vs. `/etc/cyphornips/rules/local/` (yours — never touched by an update).
- **64‑character SHA‑256 canonical fingerprints**, normalized across protocol, direction, addresses, ports, and options — so two rules sharing the same SID from different sources are treated as independent, both load, and standard Action Precedence (`PASS > ALERT`) applies between them instead of one silently deleting the other.
- Snort/Suricata‑compatible rule grammar (`action proto src_ip src_port direction dst_ip dst_port (options...)`), deliberately with **no `$HOME_NET`‑style variables** — addresses are always literal or CIDR, with optional `!` exclusion.
- Four actions: `alert` (log only), `drop` (silent discard + kernel eBPF entry), `reject` (discard + active TCP RST / ICMP Port Unreachable), `pass` (bypass, always wins).

### Packet I/O, Offloads & Pre‑Forward Policy Synchronization
- **Userspace pre‑forward policy sync:** before calling `packet_io_send()`, userspace checks the packet against the same eBPF policy map the kernel classifier uses, so a packet already known to be dropped is never handed to the kernel for transmission in the first place.
- Mirrors the kernel's lookup hierarchy exactly: up to 10 bounded wildcard stages for IPv4 TCP/UDP, 4 stages for IPv4 ICMP, 1 exact 128‑bit lookup for IPv6 — zero heap allocations, zero map iterations, zero locks.
- Boot‑time TTL semantics (`CLOCK_BOOTTIME`) matching the kernel's `bpf_ktime_get_boot_ns()`; wire‑tuple‑correct evaluation on both the LAN→WAN and WAN→LAN views.
- Root cause this eliminated: **spurious `sendto()` ENOBUFS** from the kernel TC egress filter shooting down already‑policy‑dropped frames that userspace kept trying to transmit. Production measurement: **1,405 spurious ENOBUFS events / 10 min → 0**, with **390 packets** now cleanly intercepted in userspace before ever reaching the NIC, and legitimate traffic (33,291 frames / 25.5 MB) forwarded with zero loss.
- Dedicated counters (`policy_sync_drops`, `policy_sync_drops_wan`, `policy_sync_drops_lan`) and rate‑limited proof logging; a genuine `ENOBUFS` is still logged explicitly and is never guessed to be a policy drop.

### IPv6 Enforcement (NEW)
- The kernel classifier previously had no IPv6 policy map, so IPv6 traffic was inspected and alerted on but **never actually enforced**. v1.5.0 adds the `cyphorn_policy_v6` BPF map, IPv6 header parsing in the classifier, TTL‑based expiry, garbage collection, and address‑family tracking so QUIC (UDP/IPv6) flows tear down enforcement through the matching API.

### Update Engine & Atomic Pipeline
- **Decoupled provider abstraction:** GitHub CDN (`CyphornIPS/cyphorn-rules`) or a generic HTTP/S3/R2 endpoint via `--manifest-url`.
- **Ed25519‑signed manifests**, verified by default against the pinned key at `/etc/cyphornips/update-signing.pub`; unsigned/invalid manifests are rejected with zero changes to disk (`--allow-unsigned-manifest` exists for lab use only).
- **SHA‑256 artifact verification**, **combined sandboxed pre‑validation** (candidate rules merged with active local rules and test‑compiled before going live), **automated backup + rollback** if compilation or hot reload fails.
- `cyphornctl update migrate` safely restructures an old flat rules directory into `managed/` + `local/` by canonical fingerprint, with a backup taken first.

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                 13-Stage Atomic Update Pipeline & Rollback                 │
│                                                                            │
│ 1. Provider Check ─────────── GitHub CDN, or Generic HTTP / S3 / R2        │
│         │                                                                  │
│         ▼                                                                  │
│ 2. Manifest Fetch & Ed25519 Signature Verify                               │
│         │                                                                  │
│         ▼                                                                  │
│ 3. Version Comparison Gate ── (bypassed only with --force)                 │
│         │                                                                  │
│         ▼                                                                  │
│ 4. Staging Download ───────── /var/lib/cyphornips/updates/staging/         │
│         │                                                                  │
│         ▼                                                                  │
│ 5. SHA-256 Integrity Verification ── reject immediately on mismatch        │
│         │                                                                  │
│         ▼                                                                  │
│ 6. Combined Sandboxed Pre-Validation ── merged with active local rules     │
│         │                                                                  │
│         ▼                                                                  │
│ 7. Backup Snapshot ───────── managed/  →  updates/backup/                  │
│         │                                                                  │
│         ▼                                                                  │
│ 8. Atomic Installation ───── managed/ only, local/ never touched           │
│         │                                                                  │
│         ▼                                                                  │
│ 9. Ownership Reconciliation ── re-index SHA-256 canonical fingerprints     │
│         │                                                                  │
│         ▼                                                                  │
│10. Zero-Downtime Hot Reload ── via /run/cyphornips/control.sock            │
│         │                                                                  │
│         ▼                                                                  │
│   ┌─────────────┴─────────────┐                                            │
│   ▼                           ▼                                            │
│ Reload accepted          Reload rejected                                   │
│   │                           │                                            │
│   ▼                           ▼                                            │
│ New generation live     Automated Rollback:                                │
│ zero packet loss        restore backup snapshot,                           │
│                         reload the restored copy                           │
└────────────────────────────────────────────────────────────────────────────┘
```

### Zero‑Downtime Hot Reload
- Rules, threat datasets, and both policy files use **generation counters**: a reload compiles the new version into a separate buffer, validates it, and only then atomically swaps the active pointer — the running engine is never left reading a half‑updated set, and a reload that fails validation never swaps in.
- Triggered via `cyphornctl reload {rules|intelligence|domain-policy|country-policy|all}` over `/run/cyphornips/control.sock`, or POSIX signals.

### Administrative CLI (`cyphornctl`)

| Command | Purpose | Side‑effect |
|---|---|---|
| `cyphornctl status` / `status json` | Live engine metrics & generations | Read‑only |
| `cyphornctl ping` | Unix control socket health check | Read‑only |
| `cyphornctl explain <ip> <domain>` **(NEW)** | Why was this device blocked/allowed? | Read‑only |
| `cyphornctl update check` | Query remote provider for updates | Read‑only |
| `cyphornctl update migrate` | Migrate flat rules dir → `managed/` + `local/` | Restructures rules safely |
| `cyphornctl update rules` | Download, validate, install & reload rules | Updates `managed/rules.rules` |
| `cyphornctl update intelligence` | Download, validate, install & reload File Intelligence | Updates the JSON dataset |
| `cyphornctl update all` | All‑or‑nothing atomic update | Updates both datasets |
| `cyphornctl reload rules` / `intelligence` / `domain-policy` / `country-policy` / `all` | Hot‑reload from local disk | Zero‑downtime pointer swap |

```text
Options:
  --channel <name>             Update channel (currently: stable)
  --manifest-url <url>         Generic HTTP provider pointing at a manifest URL
  --repo <owner/repo>          Override GitHub provider repository
  --branch <branch>            Override GitHub provider branch
  --signing-key <path>         Ed25519 public key manifests are verified against
  --allow-unsigned-manifest    Skip manifest signature verification (lab/test only)
  --force                      Reinstall even if already at latest version
  --json                       Format output as JSON
  -s, --socket <path>          Override Unix control socket path
```

### Companion Diagnostics
- **`scripts/diagnose_client_block.sh <client-ip> [seconds]` (NEW):** captures the eBPF policy map (before/after), packet traces on both WAN and LAN, and the relevant `alerts.log`/`rules.log` window for one client into a single output directory — everything needed to root‑cause a block in one place instead of four logs and a map dump.

### Interactive Terminal Dashboard
- Full‑screen curses UI with 4 screens: **System Overview**, **Packet Flow**, **Connection Table**, **Rule Inspector** — live event feed, connection table, alert history, matched‑only rule filtering.
- Navigation: `Tab` / `1`–`4` switch screens, `Up`/`Down` scroll, `Enter` inspects the selected row, `q` exits cleanly.

### Three‑Stream Async Logging & Telemetry
- Dedicated non‑blocking writer threads for `cyphornips.log`, `rules.log`, and `alerts.log` (bounded 4,096‑entry ring buffers per stream) — a disk stall increments a drop counter instead of stalling the packet‑forwarding hot loop.
- Post‑NAT client attribution (`CLIENT=`, fixed/added this release) and ICMP correlation on every relevant log line.
- Telemetry status badges per rule: `[IDLE]`, `[MATCHED]`, `[EFFECTIVE]`, `[OVERRIDDEN]`/`OVRD`, `[DROPPED]`, `[ALERTED]` — so "matched" is never confused with "actually decided the verdict."

---

## ⚖️ Detection & Enforcement (Action Precedence)

When several rules or policies match the same packet, CyphornIPS resolves the conflict the same way every time:

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                Detection & Enforcement — Action Precedence                 │
│                                                                            │
│  Packet matched by multiple rules and/or policies simultaneously           │
│                              │                                             │
│                              ▼                                             │
│                  ┌───────────────────────┐                                 │
│                  │  Any PASS match?      │                                 │
│                  └───────────┬───────────┘                                 │
│                     yes ─────┤───── no                                     │
│                      │                │                                    │
│                      ▼                ▼                                    │
│        ┌─────────────────────┐   ┌───────────────────────┐                 │
│        │ PASS — forward now  │   │ Any DROP/REJECT match?│                 │
│        │ overrides DROP &    │   └───────────┬───────────┘                 │
│        │ ALERT completely    │      yes ──────┤────── no                   │
│        └─────────────────────┘       │               │                     │
│                                       ▼               ▼                    │
│                       ┌──────────────────────┐  ┌──────────────────┐       │
│                       │ DROP/REJECT — discard │  │ Any ALERT match? │      │
│                       │ inline + install eBPF │  └────────┬─────────┘      │
│                       │ drop entry            │    yes ───┤─── no          │
│                       └──────────────────────┘      │            │         │
│                                                      ▼            ▼        │
│                                        ┌───────────────────┐  ┌────────┐   │
│                                        │ ALERT — forward & │  │ Forward│   │
│                                        │ log to alerts.log │  │ no log │   │
│                                        └───────────────────┘  └────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

| Badge | Meaning | Verdict | Internal accounting |
|---|---|:---:|---|
| `[IDLE]` | Loaded, never matched a packet | Neutral | `matched == 0`, `effective == 0` |
| `[MATCHED]` | Matched traffic at least once | Contextual | `matched_packets > 0` |
| `[EFFECTIVE]` | Determined the final verdict | Definitive | `effective_matches > 0` |
| `[OVERRIDDEN]` (`OVRD`) | Matched, but superseded by a higher‑priority action | Allowed | `matched > 0`, `effective == 0` |
| `[DROPPED]` | Inline blocking rule actively dropped packets | **BLOCKED** | Increments `dropped_packets` |
| `[ALERTED]` | Generated security alerts | Allowed | Increments `alert_count` |

---

## 🔍 Domain Policy — "Why Was I Blocked?"

```text
cyphornctl explain <client-ip> <domain>
│
├── 1. Resolve "<domain>" to its configured category
│
├── 2. Collect every policy scope defined for that category
│
└── 3. Rank every scope by specificity — highest tier always wins
    │
    ├── Tier 3 — Exact host        e.g. 192.168.1.50 / fe80::1
    ├── Tier 2 — CIDR subnet       e.g. 192.168.1.0/24 (longer prefix wins ties)
    └── Tier 1 — Wildcard          ANY / * / all   (fallback only)
        │
        └── Highest tier PRESENT FOR THIS CLIENT decides the tier.
            │
            ├── More than one rule in that tier?
            │       └── yes → highest PRIORITY value wins
            │
            └── Exactly one rule in that tier?
                    └── that single rule is the verdict — priority irrelevant
                │
                ▼
        Prints to terminal:
          • BLOCKED/ALLOWED <client> -> <domain>
          • the deciding scope, and why it beat the others
          • one ready-to-paste config line to change the outcome
          • a table: SCOPE | SAYS | PRIORITY | APPLIES TO THIS DEVICE
          • what this answer does NOT cover (country policy, signature
            rules, already-installed kernel entries)
```

Sample output:

```text
$ cyphornctl explain 192.168.100.53 facebook.com

  BLOCKED  192.168.100.53  ->  facebook.com
  --------------------------------------------------

  Why
    'facebook.com' is in the category 'social_media'.
    The rule for 192.168.100.0/24 says BLOCK, and it is the most
    specific rule that covers this device.

  To allow it
    Add this line to /etc/cyphornips/domain_policy.conf:

        policy social_media 192.168.100.53 ALLOW 100

    Then: cyphornctl reload domain-policy

  Every rule for category 'social_media'
    SCOPE                  SAYS   PRIORITY  APPLIES TO THIS DEVICE
     ANY                    BLOCK  5         yes, but a more specific rule wins
  -> 192.168.100.0/24       BLOCK  10        yes - this one decides
     192.168.100.50         ALLOW  100       no

  Not covered by this answer
    Country policy: active, and judges the destination country. That depends
                    on the address this name resolves to, so it cannot be
                    decided from the name alone.
    Signature rules: N loaded; any of them may also match this traffic.
    Datapath:        entries already installed in the kernel are not read here.
```

The table is deliberately the part that teaches: an `ALLOW` at priority 100 losing to a `BLOCK` at priority 10 looks wrong until it's visible that **specificity is ranked before priority**, and that the `ALLOW` doesn't even cover this device. `cyphorn_domain_policy_explain` reimplements the ranking independently of the live evaluator's code path specifically so the test suite can assert the two never drift apart — an explanation that disagrees with the real decision would be worse than no explanation at all. Output is colored only when writing to a terminal, so piped output stays greppable. Country policy, signature rules, and already‑installed kernel entries are explicitly named as **out of scope** for this command rather than silently ignored.

---

## 🔐 Security Hardening & Performance

- **Compiler hardening:** `-O2 -D_FORTIFY_SOURCE=2`, `-fstack-protector-strong`, `-fPIE`/`-pie`, `-Wl,-z,relro,-z,now`, `-Wformat -Wformat-security`.
- **Privilege separation:** steady‑state capability set is `CAP_NET_ADMIN CAP_NET_RAW CAP_BPF CAP_SYS_RESOURCE CAP_PERFMON CAP_SYS_ADMIN` — never full root. The control socket is root/admin‑group only, mode `0660`.
- **Cryptographic integrity:** Ed25519 manifest signatures, SHA‑256 payload hashes, isolated dry‑run compilation before anything reaches the running engine.
- **eBPF's own safety guarantees:** the kernel verifier bounds‑checks every pointer and array index before the classifier is allowed to attach; loops are bounded; memory access is limited to the explicit `cyphorn_policy` maps.
- **Host self‑protection:** at startup the engine enumerates its own local IP addresses and refuses to let any rule or policy drop or reset traffic aimed at the appliance's own management IPs, SSH, control socket, or diagnostic API — a bad rule can't lock you out of the box protecting the network.
- **In‑kernel drop maps:** `cyphorn_policy` (IPv4) and `cyphorn_policy_v6` (IPv6, **new this release**) key on address → `expire_ns`, giving an early `TC_ACT_SHOT` drop in the TC ingress hook before the IP stack or user‑space ever sees the packet.
- **A single hot loop, atomic reload:** packet capture, decoding, and rule matching run in one straight‑line loop with no inter‑packet locking; async log writers, the conntrack cache listener, and the diagnostic API server all run on their own background threads so nothing can stall the datapath.

---

## 🧪 Testing & Verification

CyphornIPS ships with an extensive regression suite covering protocol decoders, sticky buffers, file reassembly, threat intelligence, action precedence, rule ownership, update staging, automated rollback, packet I/O offload management, userspace pre‑forward policy synchronization, and — new this release — SMTP/email parsing, per‑client policy scoping under NAT, and IPv6 enforcement.

```bash
./tests/run_all_tests.sh
```

**Key subsystem verification:**
- **Policy Synchronization (`tests/test_policy_synchronization.c`):** exact 5‑tuples, all 10 BPF wildcard stages, ICMP's 4‑step hierarchy, IPv6 single‑lookup semantics, boot‑time TTL expiration, directional wire‑header matching, forwarding drop invariants, and `eth0` hardware isolation.
- **Packet I/O & Offload Management (`tests/test_packet_io.c`):** dual‑interface initialization, non‑blocking sockets, MTU enforcement, dynamic ethtool GRO discovery, `policy_sync_drops`/`ENOBUFS` diagnostic accounting.

Manual verification against a running engine:

```bash
curl -i -s -m 3 http://198.51.100.10/test                                            # HTTP inline drop / alert
curl -i -s -k -m 3 --resolve facebook.com:443:198.51.100.20 https://facebook.com/    # HTTPS SNI domain policy
nc -z -v -w 3 198.51.100.10 443                                                       # TCP handshake / reset
ping -c 3 198.51.100.10                                                               # ICMP filtering
dig @198.51.100.1 malware-domain.test A                                               # DNS query filtering

tail -f /var/log/cyphornips/alerts.log
cyphornctl explain 192.168.1.50 facebook.com
cyphornctl status
```

---

## ⚠️ Known Limitations

- **No TLS decryption (MITM):** CyphornIPS never terminates a session, forges a CA certificate, or decrypts application data — only unencrypted handshake metadata (SNI, JA3, X.509 fields, negotiated version) is read.
- **QUIC scope:** only the Initial packet's CRYPTO frame (TLS 1.3 ClientHello) can be recovered; 1‑RTT application data uses session keys CyphornIPS has no way to derive.
- **Asymmetric routing:** stateful TCP reassembly, flowbits, and protocol tracking require both directions of a flow to cross the *same* bridge instance.
- **Topology:** dual‑interface inline appliance — scaling across multiple chassis requires an upstream Layer 3 load balancer; there's no built‑in clustering.
- **SMB / NFS:** rule syntax is recognized by the parser, but there's no runtime extraction adapter yet — such rules parse but never fire.
- **HTTP/2:** binary multiplexed frame decoding is on the roadmap, not in this release.

---

## ⚠️ Disclaimer & Operational Notice (إخلاء المسؤولية)

> [!WARNING]
> **CyphornIPS v1.5.0 is currently an experimental Developer Preview / Alpha release.** It is provided for evaluation, security research, lab validation, and community testing. **It is not designed, certified, or intended to serve as a sole, unmonitored security perimeter in mission‑critical production environments.**

- **Nature of the software:** a network security monitoring and intrusion prevention tool — **not** an absolute or infallible guarantee against cyber attacks, malware, data breaches, or unauthorized intrusions.
- **No absolute detection warranty:** efficacy depends on signature coverage, heuristic accuracy, protocol parser fidelity, network throughput, and release version.
- **False positives & false negatives:** signatures may flag legitimate traffic or miss novel, obfuscated, or zero‑day vectors.
- **Mandatory policy staging:** administrators are solely responsible for testing rules and domain policies in a non‑production environment before enabling inline blocking.
- **Operational impact:** misconfigured `DROP` rules can break legitimate workflows; overly broad `PASS` rules can whitelist unauthorized traffic.
- **Defense‑in‑depth:** CyphornIPS does **not** replace host hardening, patching, secure backups, IAM/MFA, network micro‑segmentation, or incident response.
- **Limitation of liability:** provided "AS‑IS" without warranty of any kind; authors and contributors are not liable for damages arising from use or misconfiguration.
- **Regulatory compliance:** you are responsible for ensuring packet inspection, file extraction, and logging comply with applicable local privacy and telecommunications law.

**Case study — why `content:"admin"` is not a safe pattern:**

```text
drop tcp any any -> any 80 (msg:"Block Web Admin"; http.uri; content:"admin"; sid:9900001; rev:1;)
```

This also matches `/assets/js/admin-tools.min.js`, `/portal/readmission/status`, and `/documentation/administrator-guide.html` — all legitimate. **Always deploy new rules as `alert` first**, watch match rates on the dashboard's Rule Inspector, and only promote to `drop`/`reject` once verified.

### Golden Rules of CyphornIPS Administration

1. **Never** manually edit files inside `rules/managed/` — upstream updates own this directory exclusively; put custom rules in `rules/local/`.
2. **Never** rely on SID alone for rule identity — CyphornIPS uses the 64‑character Canonical Fingerprint to safely support shared SIDs across upstream and local rule sets.
3. **Never** delete `rules/local/` while troubleshooting — local rules and whitelist overrides survive update rollbacks intact.
4. **Never** assume `[MATCHED]` means traffic was dropped — check `[EFFECTIVE]` and `[DROPPED]` to see whether a higher‑priority `PASS` overrode it.

---

## 📚 Documentation Suite & References

- 📖 **[CyphornIPS Full Technical Documentation](https://cyphornips.github.io/cyphorn-engine/cyphornips-documentation.html)** — comprehensive single‑page guide covering every subsystem, decoder, sticky buffer, and operational workflow.
- 📂 **[Modular Documentation Suite](https://cyphornips.github.io/cyphorn-engine/docs/)**:
  - 💻 [CLI Reference](docs/cli-reference.md)
  - 🛡️ [Rule Engine & Fingerprinting](docs/rules.md)
  - ⚖️ [Rule Action Precedence](docs/action-precedence.md)
  - 📊 [Telemetry & UI Metrics](docs/telemetry.md)
  - 🔄 [Update Engine & Providers](docs/updates.md)
  - 🛠️ [Troubleshooting Guide](docs/troubleshooting.md)
  - 🏗️ [Developer & Systems Architecture](docs/architecture.md)

> **GeoIP data attribution:** the bundled country/ASN databases (`/etc/cyphornips/geoip/`) are provided by **[DB-IP.com](https://db-ip.com)** — *IP Geolocation by DB‑IP* — licensed under the CyphornIPS Proprietary License, Copyright (c) 2026 CyphornIPS. A copy of this notice is also seeded as `/etc/cyphornips/geoip/ATTRIBUTION.txt`.

---

## ☕ Support CyphornIPS 

CyphornIPS is independently developed network security software.

If you find CyphornIPS useful and would like to support its continued development, you can support the project here:

[☕ Support Cyphorn Development](https://www.buymeacoffee.com/malhummada)


---

