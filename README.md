<div align="center">

---

# 🛡️ CyphornIPS v1.8.0

### High-Performance Inline Network Intrusion Prevention System

---

</div>

[![Language: C17](https://img.shields.io/badge/Language-C17-00599C.svg)](https://en.wikipedia.org/wiki/C17_(C_standard_revision))
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux%20Kernel%205.8%2B-FCC624.svg?logo=linux&logoColor=black)](https://kernel.org)
[![eBPF / TC](https://img.shields.io/badge/Kernel-eBPF%20%2F%20TC%20Filter-red.svg)](https://ebpf.io/)
[![Architectures](https://img.shields.io/badge/Arch-arm64%20%7C%20amd64-64748b.svg?logo=debian&logoColor=white)](#-installation)
[![Verification](https://img.shields.io/badge/Verification-2%2C568%20checks%20passing-10b981.svg)](#-testing--verification)
[![False Positives](https://img.shields.io/badge/False%20Positives-0%20of%2046%20benign-22c55e.svg)](#-false-positive-budget)
[![Evasion](https://img.shields.io/badge/Evasion%20Corpus-11%20of%2011-0ea5e9.svg)](#-evasion-resistance)
[![Updates](https://img.shields.io/badge/Updates-Ed25519%20%2B%20SHA--256-a855f7.svg)](#-signed-updates)
[![Docs](https://img.shields.io/badge/Manual-English%20%2B%20Arabic-06b6d4.svg)](https://cyphornips.github.io/cyphorn-engine/)
[![License](https://img.shields.io/badge/License-Free%20for%20personal%20use-f59e0b.svg)](LICENSE)

---

## 📖 Overview

**CyphornIPS** is a transparent, inline Intrusion Prevention System and Network
Security Monitoring engine for Linux gateways and edge firewalls. Written in C17
with native eBPF/TC kernel acceleration, it inspects up to sixteen links and
performs deep packet inspection, stateful TCP stream reassembly, in-memory file
extraction, cryptographic hash threat-intel matching, domain and country policy
enforcement, and atomic zero-downtime hot reload — all inline, at wire speed.

An intrusion prevention system asks you to trust it with every packet on your
network. That trust should be earned by evidence rather than by a feature list,
and this project is built so the evidence exists: the engine publishes a
**coverage equation** that must balance, its verification suite reports **2,568
individual checks**, and every claim on this page links to the gate that measures
it.

### ✨ What's new in v1.8.0

| | |
|---|---|
| ⚡ **Multi-threaded inspection** | one worker per core, less one, derived from the machine — [measured](#-multi-threaded-inspection), not asserted |
| 🩺 **`cyphornctl diagnose`** | finds what is stopping the appliance and prints the command that fixes it — **works while the engine is down** |
| 🔻 **`cyphornctl logs threats`** | what was blocked or flagged, in columns, repeats folded |
| 🔄 **Threat feeds that refresh** | the refresh script shipped in an earlier build and nothing ever ran it; a daily timer now does |
| 🌍 **Country policy that works** | the GeoIP path pointed at a directory the package does not install, so every country rule ever written had been inert |
| 🧵 **Enforcement on every link** | through earlier releases only the first port on each side could block; the rest were alert-only |

---
> 📘 **Looking for the complete technical reference?**
> [**CyphornIPS Full Documentation — English & Arabic**](https://cyphornips.github.io/cyphorn-engine/)
---

> [!IMPORTANT]
> **This release is ready for production use — under two conditions, and they are
> not formalities.** Watch it, and do not make it your only control. See
> [Running it in production](#-running-it-in-production) before you deploy inline
> blocking.

---

## 📦 Installation

Two packages are published. **Install the one that matches your machine** — the other will install and its binaries will not run.

```bash
dpkg --print-architecture          # prints: arm64  or  amd64
```

### 🔹 arm64 / aarch64

Banana Pi R4, Raspberry Pi, most small routing boards.

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_arm64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS      # must print OK
sudo apt install ./cyphornips_1.8.0_arm64.deb
```

### 🔹 amd64 / x86_64

Servers, virtual machines, mini-PCs.

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_amd64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS      # must print OK
sudo apt install ./cyphornips_1.8.0_amd64.deb
```

> [!CAUTION]
> **A digest that does not match means the file is not the one that was published. Do not install it.** Download again; if it differs a second time, the problem is not your network.

### 🔹 First start

```bash
sudo nano /etc/cyphornips/cyphornips.conf     # set wan_interface and lan_interface
sudo systemctl enable --now cyphornips
cyphornctl diagnose
```

A fresh install deliberately does **not** start the service: the interfaces have not been named yet, and starting would crash-loop.

### 🔹 System requirements

| Requirement | Value | Why |
|---|---|---|
| **Kernel** | **≥ 5.8** (6.x preferred) | eBPF `clsact` hooks, BPF maps, conntrack, `TPACKET_V3`, `PACKET_FANOUT` |
| **Interfaces** | at least 2, **up to 16** | physical or virtual — virtio, veth, bridge ports all work |
| **Privileges** | root, or capabilities | `CAP_NET_ADMIN` `CAP_NET_RAW` `CAP_BPF` `CAP_SYS_RESOURCE` `CAP_PERFMON` |
| **Memory** | 512 MB min, 2 GB+ recommended | limits are derived from the machine, not compiled in |
| **Storage** | 1 GB free | logs, rotated archives, extracted files, feeds |
| **`/tmp`** | mounted `1777` | a missing `mode=1777` in `/etc/fstab` silently breaks everything that writes a temporary file |

Section 0 of the manual has a thirty-second check script and the three conditions that let the engine start while it is silently wrong.

---

## 🏗️ How it proves what it inspected

Most inspection engines tell you what they found. This one also tells you what it **missed** — and refuses to let that number go unexplained.

Every frame the appliance handled must resolve to exactly one terminal state: inspected and allowed, inspected and dropped, or uninspected with a named reason. The sum is checked continuously against what the interfaces themselves report:

```
nic = prefilter + kernel_dropped + capture_dropped + engine_received
    + skipped_egress + in_flight
```

When it does not balance, health says `FAILED` and `cyphornctl status` prints the residual.

> [!NOTE]
> That equation is not decoration. During development it caught **eight separate defects** that would otherwise have shipped silently — including packets counted as inspected while the engine had declined to forward them. **A blind spot you can measure is a different thing from one you cannot.**

---

## 🧩 Complete feature set

| | |
|---|---|
| **Inline enforcement** | eBPF/TC classifier in the kernel's own path — a drop is a drop, not a log line |
| **Multi-threaded inspection** | one worker per core, less one, derived from the machine |
| **Up to 16 links** | all inspected, all enforced on |
| **Protocol parsers** | TCP reassembly with overlap policy, IP defragmentation, HTTP, TLS (SNI, JA3, certificate fields), DNS, QUIC, SMTP, FTP, IKE, ICMP, ICMPv6 |
| **File intelligence** | in-memory extraction over HTTP and FTP, MD5/SHA-1/SHA-256/SHA3-384, magic-byte classification |
| **Domain & country policy** | category blocking with CIDR scoping and NAT client resolution; per-country allow/drop by direction |
| **Rules** | Suricata-compatible syntax, flowbits, datasets, hot-reloadable without dropping a packet |
| **Signed updates** | Ed25519 verifies the manifest, SHA-256 binds each artifact — **no phone-home** |
| **Fail mode** | `closed` or `open`, applied even after SIGKILL or power loss |
| **Self-diagnosis** | `cyphornctl diagnose` works *while the engine is down*, and every finding carries the command that fixes it |

### 🔻 Threat intelligence

Rebuilt **daily** by `cyphornips-feeds.timer` from public sources. Orders of
magnitude on a running appliance — exact counts change with every refresh, which
is the point of refreshing them:

| Source | Indicators |
|---|---|
| SSL blacklist certificates | ~10,000 |
| ThreatFox C2 domains | ~4,500 |
| URLhaus malware hostnames | ~3,500 |
| MalwareBazaar file hashes | ~1,300 |
| Malicious JA3 fingerprints | ~100 |
| **Detection rules loaded** | **12,093** |

`cyphornctl status` prints what *your* appliance has loaded right now, which is
the only count that matters to you.

> [!NOTE]
> An infrastructure allowlist keeps a hostname that merely *served* malware from
> blocking `github.com` or `drive.google.com`. A feed entry is evidence that
> something bad was hosted somewhere, not that the host is malicious.

### ⚡ Multi-threaded inspection

Measured on a four-core appliance under 4096 concurrent flows:

| Workers | Frames inspected | CPU per core |
|---|---|---|
| 1 | **65%** | 55% / 0 / 0 |
| 3 | **100%** | 28.6% / 13.3% / 13.6% |

The kernel does the splitting through an eBPF program that normalises the flow tuple, so **both directions of a conversation reach the same worker**. That is a correctness requirement, not a tuning choice: half a TCP stream reassembled twice is worse than not inspecting it.

### 🔐 Signed updates

The signature authenticates the *manifest*; the digest binds each *artifact* to it. A signed manifest with an unverified payload authenticates a promise, not a file.

```
Provider → Manifest → Ed25519 verify → Version gate → Staging
        → SHA-256 verify → Sandbox validation → Backup
        → Atomic install → Hot reload → Success? → Live / Rollback
```

No update is installed without both checks passing. There is no phone-home and no telemetry leaving the appliance.

---

## 🧪 Testing & Verification

CyphornIPS carries **99 gates** — each a script or a binary that checks one
property and prints every check on its own line. **97 pass, reporting 2,568
individual checks.**

> [!TIP]
> These are the figures from the run that produced this release, not a stored
> summary. Every row below links a claim to the gate that measures it, and the
> raw output is quoted further down.

### 🔻 Headline results

| | |
|---|---|
| ✅ **2,568** individual checks passing across the suite |
| ✅ **19 of 19** held-out attack cases detected — **100%**, against a 95% target |
| ✅ **0 of 46** benign cases produced a false positive |
| ✅ **11 of 11** evasion techniques defeated |
| ✅ **51** packet captures replay byte-identically, three runs each |
| ✅ **153** configuration keys the parser accepts, all documented |

### 🔻 Detection, measured against a held-out corpus

| Gate | What it measures | Result |
|---|---|---|
| `attack-corpus` | detection against attacks the rules were **never tuned on** | **19 / 19** ✅ |
| `fp-budget` | an adversarial benign corpus, looking for false alarms | **0 / 46** ✅ |
| `rule-quality` | every blocking rule carries the metadata to be auditable | pass ✅ |

> [!NOTE]
> **The corpus is split, and the split is the point.** Detection is measured only
> against the **held-out** half. The tuning half is measured separately and its
> figure is explicitly **not publishable** — a detection rate quoted against the
> data you tuned on measures your memory, not your engine.

### 🔻 Evasion resistance

Eleven techniques that hide an attack from an inspection engine. Each is the same
payload, delivered so that a naive parser misses it:

| Technique | Delivered as | |
|---|---|---|
| TCP segmentation | split across segments | ✅ |
| TCP segmentation | one byte per segment | ✅ |
| Retransmission | overlapping retransmits | ✅ |
| Overlap | conflicting overlapping data | ✅ |
| Out-of-order delivery | reordered segments | ✅ |
| IPv4 fragmentation | fragmented | ✅ |
| IPv4 fragmentation | fragments reordered | ✅ |
| IPv6 fragmentation | fragmented | ✅ |
| Malformed packets | malformed, then valid | ✅ |
| Encoding variation | encoded URI | ✅ |
| Protocol ambiguity | ambiguous protocol | ✅ |

Plus a control: the plain delivery is detected **exactly once**, so a technique
cannot "pass" by making the engine alert twice.

### 🔻 Correctness under adversarial input

The largest gates by number of checks — these are where a parser bug becomes a
bypass:

| Gate | Checks | What it covers |
|---|---|---|
| `determinism` | **257** | the same capture must produce the same verdict, byte for byte |
| `pipeline` | **210** | the inspection path end to end |
| `defrag` | **127** | IP fragmentation, including overlap and reordering |
| `active-response` | **113** | what the engine sends when it blocks |
| `overlap-policy` | **99** | conflicting TCP overlaps, resolved by a stated policy |
| `flag-anomaly` | **93** | TCP flag combinations that should never occur |
| `resource-limits` | **92** | behaviour at the memory and connection ceilings |
| `decoding` | **86** | L2–L4 decoders against malformed input |
| `coverage` | **70** | the accounting equation itself |
| `reassembly` | **67** | stream reassembly and overlap resolution |

### 🔻 Threading changes no verdict

| Gate | What it proves | |
|---|---|---|
| `worker-equivalence` | the same captures split 2 and 4 ways produce **identical verdicts** | ✅ |
| `flow-fanout` | both directions of a conversation reach the same worker | ✅ |
| `worker-isolation` | per-worker state never leaks between threads | ✅ |
| `race-detection` · `sanitize` | ThreadSanitizer and AddressSanitizer over the inspection path | ✅ |

### 🔻 Supply chain and operations

| Gate | What it proves | |
|---|---|---|
| `update-authenticity` | a signed manifest is accepted; one altered byte is refused — 22 cases | ✅ |
| `crash-safety` | a crash is reported on the next run rather than swallowed | ✅ |
| `fail-mode` | forwarding policy survives SIGKILL and power loss | ✅ |
| `sav` | spoofed source addresses are dropped and carry the reason | ✅ |
| `privilege-hardening` · `secret-handling` | the systemd sandbox, and what never reaches a log | ✅ |
| `server-portability` | same conclusions on links named `eno1`, `enp3s0`, `ens18` | ✅ |
| `hardware-limits` | rule ceiling, ring geometry, memory cap derived from the machine | ✅ |
| `baseline-freshness` | **43** recorded limitations still hold against the code | ✅ |

### 🔻 What the output actually looks like

```text
=== T-004 evasion corpus ===
  PASS  TCP segmentation -> segmented
  PASS  TCP segmentation (one byte) -> segmented-tiny
  PASS  overlap -> overlapping
  PASS  IPv4 fragments (reordered) -> ipv4-frag-out-of-order
  PASS  control: the plain delivery is detected exactly once

=== P6-008 false-positive budget ===
        46 benign cases, 0 produced a detection
        no category produced a false positive on this corpus
  PASS  control: a rule that fires on benign traffic is seen by this measurement

=== P6-009 attack corpus and holdout ===
        HOLDOUT: 19 of 19 attack cases detected (100%)
  PASS  P6-009: HOLDOUT detection is 100%, at or above the 95% target

=== determinism ===
  PASS  d42_boundary.pcap: 3 runs byte-identical
  PASS  d42_boundary.pcap: digest carries verdict, action, sids and reason

=== worker-equivalence ===
  PASS  the shard hash is symmetric: a flow and its reply land together
  PASS  p0005_flow_flood.pcap split 2 ways: every verdict identical (5000 packets)

=== configuration reference ===
  PASS  all 153 keys the parser accepts appear in the reference
```

### 🔻 Why these numbers mean something

Two design choices, without which a passing suite proves very little:

- **Every refusal is paired with an ablation.** A gate that tests only the
  positive case still passes after the mechanism it tests has been deleted. So
  each check that an input is *refused* is paired with a check that the same
  input *succeeds* once the reason for refusal is removed. You can see this in
  the output above: every block ends with a `control:` line.
- **The corpus is split.** Detection is measured against a held-out set the rules
  were never tuned on.

### 🔻 The two that do not pass

Named here rather than omitted. Both are environmental, and both say so
themselves:

- **`transit`** measures the appliance inspecting live client traffic and needs
  traffic on an inside link to judge anything. On a quiet network it finds
  nothing to measure.
- **`performance`** measures by replaying captures offline, and offline replay is
  forced to a single worker so its output stays reproducible — so it cannot
  answer a question about concurrency however often it runs. It is the wrong
  instrument, and a new one is needed. See [Known limitations](#️-known-limitations).

### 🔻 Verify your own installation

```bash
sha256sum -c SHA256SUMS      # the package is the one that was published
cyphorn-engine --version     # 1.8.0
cyphornctl diagnose          # works while the engine is down, too
cyphornctl status            # rules loaded, coverage invariant holds
```

---

## 🚀 Running it in production

> [!IMPORTANT]
> **This release is ready for production use — under two conditions.**

### 1️⃣ Watch it

CyphornIPS is built to be watched: it publishes a coverage equation that **fails loudly** rather than a dashboard that always looks green. Use that.

```bash
cyphornctl status                     # health, coverage, per-link loss
cyphornctl logs threats -n 20         # what was blocked or flagged
cyphornctl logs threats --blocked -f  # follow enforcement live
cyphornctl diagnose                   # each finding with the command that fixes it
```

Check `health` and the coverage invariant after every change to rules, policy or interfaces. A `DEGRADED` or `FAILED` state is the engine telling you something it cannot fix by itself.

### 2️⃣ Do not make it your only control

No IPS detects everything, and this one says so in its own limitations. CyphornIPS is **one layer**. Run it alongside the rest of what a defended network needs:

- a firewall with a default-deny posture
- endpoint protection on the hosts behind it
- patching, and a way to know what is unpatched
- authenticated, monitored administrative access
- off-box log retention, so an incident is still investigable afterwards
- backups you have actually restored from

> [!WARNING]
> **Before you put it in front of anything that matters:** read `/etc/cyphornips/country-policy.conf` and `domain_policy.conf` as if they were new — country policy was inert in earlier versions and becomes live here. Know that `fail_mode = closed` means stopping the service **takes transit down** on every inspected link, by design. Stage every rule you add, and measure loss with two samples rather than one — loss is bursty, and a single reading will tell you whatever the moment says.

---

## 🔄 Upgrading from v1.5.0

> [!WARNING]
> **Country policy has never matched anything on an installed appliance, and after this upgrade it will.** The compiled default GeoIP path pointed at a working directory the package does not install, so the database was never loaded and every `drop inbound RU`-style rule you wrote has been inert since you wrote it. After upgrading they take effect — review your country policy **before** restarting.

v1.5.0 is the release before this one. v1.7.x was never published.

```bash
sudo tar czf /root/cyphornips-config-$(date +%F).tar.gz /etc/cyphornips   # back up first
sudo apt install ./cyphornips_1.8.0_<arch>.deb
sudo systemctl restart cyphornips
cyphornctl diagnose
```

Your `/etc/cyphornips/` is never part of the package payload, so an upgrade replaces only the binaries, the BPF objects and the systemd units. With `fail_mode = closed`, transit stops for the few seconds the engine is restarting. The manual has the ordered procedure and the rollback.

---

## ⚠️ Known limitations

Stated here rather than discovered later.

| | |
|---|---|
| **Latency under concurrency** | unmeasured — the existing benchmark replays offline, which is forced to a single worker, so it cannot produce a figure. **If a latency ceiling matters to you, measure it on your own load before putting the appliance in the path.** |
| **Capture ring sizing** | per host, not per port |
| **Independent review** | none; verified on few machines and few topologies |
| **TLS** | never decrypted. Works from what is unencrypted in the handshake: SNI, JA3, certificate fields, negotiated version |
| **QUIC** | limited to the Initial packet's ClientHello; 1-RTT application data uses keys the engine cannot derive |
| **Asymmetric routing** | the engine is stateful and needs both directions of a flow; split routing breaks stream reassembly |
| **Scaling** | dual-interface inline appliance, no built-in clustering |

No IPS detects 100% of threats. Expect both false positives and false negatives, and stage every rule before you rely on it.

---

## 📚 Documentation

**[📘 CyphornIPS Full Documentation](https://cyphornips.github.io/cyphorn-engine/)** — every configuration key, every command, every rule keyword, in **English and Arabic**.

It also ships with the release as `cyphornips-documentation.html`: one self-contained file, no external script, stylesheet or font, so it opens on an appliance with no route to the internet.

---

## 📄 License

Free for personal and evaluation use. **Commercial use requires a separate signed agreement** — see [LICENSE](LICENSE) §2, which defines commercial use broadly enough to include protecting your own employer's network.

If you are unsure whether your use qualifies, ask before deploying. **Asking costs nothing and is answered.**

📧 **info@cyphorn.com**

## 📬 Contact

| | |
|---|---|
| Questions, bugs, feature requests | [open an issue](https://github.com/CyphornIPS/cyphorn-engine/issues) |
| Commercial licensing and written permission | **info@cyphorn.com** |
| Security vulnerabilities | **info@cyphorn.com** — see [SECURITY.md](SECURITY.md); please do not open a public issue |

## 📝 Changelog

[CHANGELOG.md](CHANGELOG.md) — what changed in each release, what was wrong before it, and the measurement that found it.
