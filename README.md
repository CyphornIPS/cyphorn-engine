# CyphornIPS

**Inline intrusion prevention that can prove what it inspected.**

[![Release](https://img.shields.io/badge/release-v1.8.0-blue.svg)](https://github.com/CyphornIPS/cyphorn-engine/releases/tag/v1.8.0)
[![Architectures](https://img.shields.io/badge/arch-arm64%20%7C%20amd64-555.svg)](#install)
[![License](https://img.shields.io/badge/license-Proprietary%20%7C%20free%20for%20personal%20use-orange.svg)](LICENSE)

CyphornIPS sits in the path of your traffic, decides on every packet, and
enforces that decision. It is written in C and installs as a Debian package —
no compiler, no build tooling, and nothing fetched from a network while it
installs.

---

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

When it does not balance, health says `FAILED` and `cyphornctl status` shows the
residual. That equation is not decoration: during development it caught eight
separate defects that would otherwise have shipped silently, including packets
counted as inspected while the engine had declined to forward them.

**A blind spot you can measure is a different thing from one you cannot.**

---

## Features

| | |
|---|---|
| **Inline enforcement** | eBPF/TC classifier in the kernel's own path; a drop is a drop, not a log line |
| **Multi-threaded inspection** | One worker per core, less one. Measured: one thread inspected 65% of a 4096-flow flood, three inspected 100% |
| **Multiple interfaces** | Up to 16 links, all inspected and all enforced on |
| **Protocol parsers** | TCP reassembly with overlap policy, IP defragmentation, HTTP, TLS (SNI, JA3, certificate fields), DNS, QUIC, SMTP, FTP, IKE, ICMP |
| **Threat intelligence** | URLhaus hostnames, ThreatFox C2 domains and addresses, SSL blacklist certificates, MalwareBazaar file hashes — refreshed daily by a timer |
| **Domain & country policy** | Category-based domain blocking with CIDR scoping and NAT client resolution; per-country allow/drop by direction |
| **Rules** | Suricata-compatible syntax, ~8,900 curated Emerging Threats Open rules, hot-reloadable without dropping a packet |
| **Signed updates** | Ed25519 verifies the manifest, SHA-256 binds each artifact to it. No phone-home |
| **Fail mode** | `closed` or `open`, applied even after SIGKILL or power loss |
| **Self-diagnosis** | `cyphornctl diagnose` works *while the engine is down*, and every finding carries the command that fixes it |

---

## Install

A package is published per architecture. **Install the one that matches your
machine** — the other will install and its binaries will not run.

```bash
dpkg --print-architecture      # arm64 or amd64
```

<details open>
<summary><b>arm64 / aarch64</b> — Banana Pi R4, Raspberry Pi, most routing boards</summary>

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_arm64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS     # must print OK before you install
sudo apt install ./cyphornips_1.8.0_arm64.deb
```
</details>

<details>
<summary><b>amd64 / x86_64</b> — servers, VMs, mini-PCs</summary>

```bash
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/cyphornips_1.8.0_amd64.deb
curl -LO https://github.com/CyphornIPS/cyphorn-engine/releases/download/v1.8.0/SHA256SUMS
sha256sum -c --ignore-missing SHA256SUMS     # must print OK before you install
sudo apt install ./cyphornips_1.8.0_amd64.deb
```
</details>

`SHA256SUMS` carries the digest of every published package. **A digest that does
not match means the file is not the one that was published — do not install it.**

Then name your links and start it:

```bash
sudo nano /etc/cyphornips/cyphornips.conf   # set wan_interface and lan_interface
sudo systemctl enable --now cyphornips
cyphornctl diagnose
```

A fresh install deliberately does **not** start the service: the interfaces have
not been named yet, and starting would crash-loop.

### Before you install

CyphornIPS needs a Linux that can give it what it asks for. It will start and
look healthy on one that cannot, and inspect less than you think.

| | |
|---|---|
| Kernel | **≥ 5.8** (6.x preferred) — eBPF `clsact` hooks, BPF maps, `conntrack`, `TPACKET_V3`, `PACKET_FANOUT` |
| Interfaces | at least two, up to 16; physical or virtual |
| Privileges | root, or `CAP_NET_ADMIN` `CAP_NET_RAW` `CAP_BPF` `CAP_SYS_RESOURCE` `CAP_PERFMON` |
| Memory | 512 MB minimum, 2 GB+ recommended |
| `/tmp` | mounted `1777` — a missing `mode=1777` in `/etc/fstab` breaks everything that writes a temporary file, silently |

Section 0 of the manual has the full list, a thirty-second check script, and the
three things that start and are silently wrong.

---

## Documentation

**[📘 Read the manual](https://cyphornips.github.io/cyphorn-engine/)** — every
configuration key the engine accepts, every command, every rule keyword, in
English and Arabic.

It is also attached to each release as `cyphornips-documentation.html`: one
self-contained file, no external script, stylesheet or font, so it opens on an
appliance with no route to the internet.

---

## Upgrading from 1.5.0

> **Read this one.** 1.5.0 is the release before this; 1.7.x was never published.
>
> **Country policy has never matched anything on an installed appliance, and
> after this upgrade it will.** The compiled default GeoIP path pointed at a
> working directory the package does not install, so the database was never
> loaded and every `drop inbound RU`-style rule you wrote has been inert since
> you wrote it. After upgrading they take effect. **Review
> `/etc/cyphornips/country-policy.conf` before restarting into 1.8.0**, because
> traffic you have been carrying will start being dropped.

Back up your configuration first — it is the step people skip:

```bash
sudo tar czf /root/cyphornips-config-$(date +%F).tar.gz /etc/cyphornips
```

Your `/etc/cyphornips/` is never part of the package payload, so an upgrade
replaces only the binaries, the BPF objects and the systemd units. With
`fail_mode = closed`, transit stops for the few seconds the engine is
restarting. The manual has the ordered procedure and the rollback.

---

## Verification

CyphornIPS carries **99 gates**, each a script or a binary that checks one
property and prints every check on its own line. **97 pass** on the reference
appliance. They are run before a release, and these are the figures from the run
that produced this one — not a stored summary.

| Gate | What it proves | Result |
|---|---|---|
| `determinism` | **51** packet captures replay byte-identically, three runs each | pass |
| `worker-equivalence` | the same captures split 2 and 4 ways produce identical verdicts — threading changes no decision | pass |
| `flow-fanout` | both directions of a conversation reach the same worker, so a TCP stream is never reassembled twice | pass |
| `attack-corpus` | **19 of 19** held-out attack cases detected, and **0 of 10** benign cases flagged | pass |
| `update-authenticity` | a signed manifest is accepted and a manifest with one byte altered is refused — 22 cases, each refusal paired with an ablation | pass |
| `config-reference` | all **153** configuration keys the parser accepts are documented | pass |
| `coverage` | every frame resolves to exactly one terminal state and the equation balances | pass |
| `baseline-freshness` | **43** recorded limitations still hold against the code — a claim that became false is caught | pass |
| `server-portability` | the engine reaches the same conclusions on links named `eno1`, `enp3s0`, `ens18` as on the appliance's own | pass |
| `hardware-limits` | rule ceiling, ring geometry and memory cap are derived from the machine, not compiled in | pass |
| `race-detection`, `sanitize` | ThreadSanitizer and AddressSanitizer over the inspection path | pass |
| `privilege-hardening`, `secret-handling` | the systemd sandbox and what never reaches a log | pass |

**The two that do not pass are environmental, and both say so themselves.**
`transit` measures the appliance inspecting live client traffic and needs traffic
on an inside link to judge anything; on a quiet network it finds nothing to
measure. `performance` measures by replaying captures offline, and offline replay
is forced to a single worker so its output stays reproducible — so it cannot
answer a question about concurrency however often it runs. Neither is a defect in
the engine, and neither is hidden: see **Limits** below.

Two design choices behind those figures are worth stating, because they are what
makes a number mean something:

- **Every refusal is paired with an ablation.** A gate that only tests the
  positive case passes when the mechanism it tests has been deleted. So each
  check that something is refused is accompanied by a check that the same input
  succeeds once the reason for refusal is removed.
- **The corpus is split.** Detection is measured against a **held-out** set the
  rules were never tuned on. The tuning set is measured separately and its figure
  is explicitly not publishable.

You can verify an installation yourself without any of this — the manual's
*Testing & Validation* section has the five commands, starting with
`sha256sum -c` and `dpkg -V cyphornips`.

---

## Limits

This is a **developer preview**, and the word is meant literally.

- **Verified on few machines and few topologies.** There has been no independent
  security review.
- **Latency under concurrency is unmeasured.** The threading is measured by what
  it inspects — one worker inspected 65% of a 4096-flow flood, three inspected
  100% — but the latency the engine adds with several threads running has no
  published figure, because the existing benchmark cannot produce one. If a
  latency ceiling matters to you, **measure it on your own load before putting
  the appliance in the path.**
- **Capture ring sizing is per host, not per port.**
- No IPS detects everything. Expect false positives and false negatives, and
  stage every rule before you rely on it.

Run it at home, run it on a lab network, tell me what breaks. Do not put it in
front of a business without reading [LICENSE](LICENSE) first — and then talking
to me.

---

## License

Free for personal and evaluation use. **Commercial use requires a separate
signed agreement** — see [LICENSE](LICENSE) §2, which defines commercial
use broadly enough to include protecting your own employer's network.

If you are unsure whether your use qualifies, ask before deploying. Asking costs
nothing.

📧 **info@cyphorn.com**

---

## Contact

| | |
|---|---|
| Questions, bugs, feature requests | [open an issue](https://github.com/CyphornIPS/cyphorn-engine/issues) |
| Commercial licensing and written permission | **info@cyphorn.com** |
| Security vulnerabilities | **info@cyphorn.com** — see [SECURITY.md](SECURITY.md); please do not open a public issue |

---

## Changelog

[CHANGELOG.md](CHANGELOG.md) — what changed in each release, what was wrong
before it, and the measurement that found it. The 1.7.x entries are development
milestones that were never published; they are kept because each one records a
defect and how it was caught.
