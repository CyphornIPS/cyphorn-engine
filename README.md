<div align="center">

# CyphornIPS

**Inline intrusion prevention that can prove what it inspected.**

[![Release](https://img.shields.io/badge/release-v1.8.0-2563eb.svg)](https://github.com/CyphornIPS/cyphorn-engine/releases/tag/v1.8.0)
[![Gates](https://img.shields.io/badge/verification-97%20of%2099%20gates-10b981.svg)](#verification)
[![Arch](https://img.shields.io/badge/arch-arm64%20%7C%20amd64-64748b.svg)](#install)
[![Docs](https://img.shields.io/badge/manual-EN%20%2B%20AR-06b6d4.svg)](https://cyphornips.github.io/cyphorn-engine/)
[![License](https://img.shields.io/badge/license-free%20for%20personal%20use-f59e0b.svg)](LICENSE)

[Install](#install) · [Manual](https://cyphornips.github.io/cyphorn-engine/) · [Verification](#verification) · [Operating it](#running-it-in-production) · [License](#license)

</div>

---

CyphornIPS sits in the path of your traffic, decides on every packet, and
enforces that decision in the kernel. It is written in C and installs as a
Debian package — no compiler, no build tooling, nothing fetched from a network
while it installs.

## What makes it different

Most inspection engines tell you what they found. This one also tells you what
it **missed** — and refuses to let that number go unexplained.

Every frame the appliance handled must resolve to exactly one terminal state:
inspected and allowed, inspected and dropped, or uninspected with a named
reason. The sum is checked continuously against what the network interfaces
themselves report:

```
nic = prefilter + kernel_dropped + capture_dropped + engine_received
    + skipped_egress + in_flight
```

When it does not balance, health says `FAILED` and `cyphornctl status` prints the
residual. That equation is not decoration: during development it caught eight
separate defects that would otherwise have shipped silently, including packets
counted as inspected while the engine had declined to forward them.

**A blind spot you can measure is a different thing from one you cannot.**

---

## Install

Two packages are published. **Install the one that matches your machine** — the
other will install and its binaries will not run.

```bash
dpkg --print-architecture          # prints: arm64  or  amd64
```

### arm64 / aarch64

Banana Pi R4, Raspberry Pi, most small routing boards.

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_arm64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS      # must print OK
sudo apt install ./cyphornips_1.8.0_arm64.deb
```

### amd64 / x86_64

Servers, virtual machines, mini-PCs.

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_amd64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS      # must print OK
sudo apt install ./cyphornips_1.8.0_amd64.deb
```

> **A digest that does not match means the file is not the one that was
> published. Do not install it.**

Then name your links and start it:

```bash
sudo nano /etc/cyphornips/cyphornips.conf     # set wan_interface and lan_interface
sudo systemctl enable --now cyphornips
cyphornctl diagnose
```

A fresh install deliberately does **not** start the service: the interfaces have
not been named yet, and starting would crash-loop.

### What it needs from Linux

| | |
|---|---|
| Kernel | **≥ 5.8** (6.x preferred) — eBPF `clsact` hooks, BPF maps, conntrack, `TPACKET_V3`, `PACKET_FANOUT` |
| Interfaces | at least two, **up to 16**; physical or virtual (virtio, veth, bridge ports) |
| Privileges | root, or `CAP_NET_ADMIN` `CAP_NET_RAW` `CAP_BPF` `CAP_SYS_RESOURCE` `CAP_PERFMON` |
| Memory | 512 MB minimum, 2 GB+ recommended |
| Storage | 1 GB free |
| `/tmp` | mounted `1777` — a missing `mode=1777` in `/etc/fstab` breaks everything that writes a temporary file, silently |

Section 0 of the manual has the full list, a thirty-second check script, and the
three conditions that let the engine start while it is silently wrong.

---

## What it does

| | |
|---|---|
| **Inline enforcement** | eBPF/TC classifier in the kernel's own path — a drop is a drop, not a log line |
| **Multi-threaded inspection** | One worker per core, less one, derived from the machine |
| **Up to 16 links** | All inspected, all enforced on |
| **Protocol parsers** | TCP reassembly with overlap policy, IP defragmentation, HTTP, TLS (SNI, JA3, certificate fields), DNS, QUIC, SMTP, FTP, IKE, ICMP, ICMPv6 |
| **Threat intelligence** | Refreshed daily by a systemd timer — see the live counts below |
| **Domain & country policy** | Category blocking with CIDR scoping and NAT client resolution; per-country allow/drop by direction |
| **Rules** | Suricata-compatible syntax, hot-reloadable without dropping a packet |
| **Signed updates** | Ed25519 verifies the manifest, SHA-256 binds each artifact to it. **No phone-home** |
| **Fail mode** | `closed` or `open`, applied even after SIGKILL or power loss |
| **Self-diagnosis** | `cyphornctl diagnose` works *while the engine is down*, and every finding carries the command that fixes it |

### Threat intelligence, as loaded on a running appliance

| Source | Indicators |
|---|---|
| SSL blacklist certificates | **10,822** |
| ThreatFox C2 domains | **4,345** |
| URLhaus malware hostnames | **3,543** |
| MalwareBazaar file hashes | **1,360** |
| Malicious JA3 fingerprints | **97** |
| Detection rules loaded | **12,093** |

Rebuilt daily by `cyphornips-feeds.timer`. An infrastructure allowlist keeps a
hostname that merely *served* malware from blocking `github.com`.

### Threading, measured rather than asserted

Four-core appliance, 4096 concurrent flows:

| Workers | Frames inspected | CPU per core |
|---|---|---|
| 1 | **65%** | 55% / 0 / 0 |
| 3 | **100%** | 28.6% / 13.3% / 13.6% |

The kernel does the splitting through an eBPF program that normalises the flow
tuple, so **both directions of a conversation reach the same worker**. That is a
correctness requirement, not a tuning choice: half a TCP stream reassembled
twice is worse than not inspecting it.

---

## Verification

CyphornIPS carries **99 gates** — each a script or a binary that checks one
property and prints every check on its own line. **97 pass.** These are the
figures from the run that produced this release, not a stored summary.

| Gate | What it proves | |
|---|---|---|
| `determinism` | **51** packet captures replay byte-identically, three runs each | ✅ |
| `worker-equivalence` | the same captures split 2 and 4 ways produce identical verdicts | ✅ |
| `flow-fanout` | both directions of a conversation reach the same worker | ✅ |
| `attack-corpus` | **19 of 19** held-out attack cases detected, **0 of 10** benign flagged | ✅ |
| `update-authenticity` | a signed manifest is accepted; one altered byte is refused — 22 cases | ✅ |
| `config-reference` | all **153** configuration keys the parser accepts are documented | ✅ |
| `coverage` | every frame resolves to one terminal state and the equation balances | ✅ |
| `baseline-freshness` | **43** recorded limitations still hold against the code | ✅ |
| `server-portability` | same conclusions on links named `eno1`, `enp3s0`, `ens18` | ✅ |
| `hardware-limits` | rule ceiling, ring geometry and memory cap derived from the machine | ✅ |
| `race-detection` · `sanitize` | ThreadSanitizer and AddressSanitizer over the inspection path | ✅ |
| `privilege-hardening` · `secret-handling` | the systemd sandbox, and what never reaches a log | ✅ |

Two design choices are what make those numbers mean something:

- **Every refusal is paired with an ablation.** A gate that tests only the
  positive case still passes after the mechanism it tests has been deleted. So
  each check that an input is refused is paired with a check that the same input
  succeeds once the reason for refusal is removed.
- **The corpus is split.** Detection is measured against a **held-out** set the
  rules were never tuned on. The tuning set is measured separately and its
  figure is explicitly not publishable.

**The two that do not pass are environmental, and both say so themselves.**
`transit` measures the appliance inspecting live client traffic and needs traffic
on an inside link to judge anything. `performance` measures by replaying captures
offline, and offline replay is forced to a single worker so its output stays
reproducible — so it cannot answer a question about concurrency however often it
runs. See [Known limits](#known-limits).

### Verify your own installation

```bash
sha256sum -c SHA256SUMS      # the package is the published one
cyphorn-engine --version     # 1.8.0
cyphornctl diagnose          # works while the engine is down, too
cyphornctl status            # rules loaded, coverage invariant holds
```

---

## Running it in production

**This release is ready for production use — under two conditions, and they are
not formalities.**

**1. Watch it.** CyphornIPS is built to be watched: it publishes a coverage
equation that fails loudly rather than a dashboard that always looks green.
Use that.

```bash
cyphornctl status                    # health, coverage, per-link loss
cyphornctl logs threats -n 20        # what was blocked or flagged
cyphornctl logs threats --blocked -f # follow enforcement live
cyphornctl diagnose                  # each finding with the command that fixes it
```

Check `health` and the coverage invariant after every change to rules, policy or
interfaces. A `DEGRADED` or `FAILED` state is the engine telling you something it
cannot fix by itself.

**2. Do not make it your only control.** No IPS detects everything, and this one
says so in its own limitations. CyphornIPS is one layer. Run it alongside the
rest of what a defended network needs:

- a firewall with a default-deny posture
- endpoint protection on the hosts behind it
- patching, and a way to know what is unpatched
- authenticated, monitored administrative access — see the note below
- off-box log retention, so an incident is still investigable afterwards
- backups you have actually restored from

**Before you put it in front of anything that matters:**

- Read `/etc/cyphornips/country-policy.conf` and `domain_policy.conf` as if they
  were new. Country policy has been inert in earlier versions and becomes live
  here — see [Upgrading](#upgrading-from-150).
- Know that `fail_mode = closed` means stopping the service **takes transit
  down** on every inspected link, by design.
- Stage every rule you add. Measure loss before and after with two samples, not
  one — loss is bursty and a single reading will tell you whatever the moment
  says.
- Serve the web GUI over TLS and restrict administrative ports. The engine does
  not do this for you.

---

## Upgrading from 1.5.0

> 1.5.0 is the release before this one. 1.7.x was never published.
>
> **Country policy has never matched anything on an installed appliance, and
> after this upgrade it will.** The compiled default GeoIP path pointed at a
> working directory the package does not install, so the database was never
> loaded and every `drop inbound RU`-style rule you wrote has been inert since
> you wrote it. After upgrading they take effect.

Back up first — it is the step people skip:

```bash
sudo tar czf /root/cyphornips-config-$(date +%F).tar.gz /etc/cyphornips
```

Your `/etc/cyphornips/` is never part of the package payload, so an upgrade
replaces only the binaries, the BPF objects and the systemd units. With
`fail_mode = closed`, transit stops for the few seconds the engine is
restarting. The manual has the ordered procedure and the rollback.

---

## Known limits

Stated here rather than discovered later.

- **Latency under concurrency is unmeasured.** The threading is measured by what
  it inspects, but the latency the engine adds with several threads running has
  no published figure — the existing benchmark replays offline, which is forced
  to a single worker, so it cannot produce one. **If a latency ceiling matters
  to you, measure it on your own load before putting the appliance in the path.**
- **Capture ring sizing is per host, not per port.**
- **No independent security review.** Verified on few machines and few
  topologies.
- **No TLS decryption.** It works from what is unencrypted in the handshake:
  SNI, JA3, certificate fields, negotiated version. QUIC is limited to the
  Initial packet's ClientHello.
- **Stateful, so it needs both directions of a flow.** Asymmetric routing breaks
  stream reassembly.
- **Dual-interface inline appliance.** No built-in clustering.

---

## Documentation

**[📘 Read the manual](https://cyphornips.github.io/cyphorn-engine/)** — every
configuration key, every command, every rule keyword, in English and Arabic.

It also ships with the release as `cyphornips-documentation.html`: one
self-contained file, no external script, stylesheet or font, so it opens on an
appliance with no route to the internet.

---

## License

Free for personal and evaluation use. **Commercial use requires a separate
signed agreement** — see [LICENSE](LICENSE) §2, which defines commercial use
broadly enough to include protecting your own employer's network.

If you are unsure whether your use qualifies, ask before deploying. Asking costs
nothing and is answered.

📧 **info@cyphorn.com**

## Contact

| | |
|---|---|
| Questions, bugs, feature requests | [open an issue](https://github.com/CyphornIPS/cyphorn-engine/issues) |
| Commercial licensing and written permission | **info@cyphorn.com** |
| Security vulnerabilities | **info@cyphorn.com** — see [SECURITY.md](SECURITY.md); please do not open a public issue |

## Changelog

[CHANGELOG.md](CHANGELOG.md) — what changed in each release, what was wrong
before it, and the measurement that found it.
