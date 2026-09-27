# Security Policy — CyphornIPS

## Supported Versions

A release is supported until the release after next is published. Security
fixes are issued for supported releases only; everything else is expected to
upgrade.

| Version | Supported | Notes |
|---------|-----------|-------|
| 1.7.x   | Yes       | Current |
| 1.5.x   | Yes       | Previous release; security fixes only |
| < 1.5   | No        | Upgrade |

The engine prints the version it is actually running:

    cyphorn-engine --version
    cyphornctl status

Report the number that command prints, not the one on the package. When they
disagree, say so in the report — that disagreement is itself worth knowing.

## Reporting a Vulnerability

Report privately before disclosing publicly. Do not open a public GitHub Issue
for a security vulnerability.

**Preferred:** GitHub's private vulnerability reporting, from the Security tab
of this repository. It is encrypted in transit and at rest, keeps the thread
attached to the advisory, and needs no key exchange.

**Alternative:** info@cyphorn.com. Plain email is not confidential in
transit; for anything you would not want read by a third party, use the
GitHub channel above.

Include:

- A description of the vulnerability and its potential impact
- Steps to reproduce, or a proof-of-concept if available
- The affected version, as `cyphorn-engine --version` reports it
- Your severity assessment, if you have one — CVSS v3.1 if you use a scale,
  and plain words are fine if you do not

You will receive acknowledgment within 5 business days, and status updates at
least every 14 days until the report is resolved or declined.

## Safe Harbor for Good-Faith Security Research

The CyphornIPS License prohibits reverse engineering, decompilation, and
disassembly of the Software. That prohibition does not apply to good-faith
security research meeting these conditions:

1. Make a good-faith effort to avoid privacy violations, data destruction and
   service interruption on systems you are not authorised to test
2. Test only against your own CyphornIPS installation, in an isolated lab
3. Report privately before public disclosure
4. Do not exfiltrate or publish proprietary code, algorithms or detection
   logic beyond what is necessary to describe the vulnerability
5. Allow a minimum of 90 days to remediate before public disclosure

Research within these conditions is authorised, and CyphornIPS will not pursue
legal action over it.

## Coordinated Disclosure

| | |
|---|---|
| Day 0 | Report received; acknowledged within 5 business days |
| Day 0–90 | Investigation and fix, with status updates every 14 days |
| Day 90+ | Fix released with coordinated public disclosure |

**If 90 days pass without a fix**, you are free to disclose. We would rather
be published about than have a reporter feel trapped by a timeline we did not
meet. Tell us you intend to, so the advisory and the disclosure can at least
be simultaneous.

**CVEs.** A CVE is requested for any confirmed vulnerability in supported code
that an operator would have to act on. Reporters are credited as the CVE
reporter unless they ask otherwise.

## Scope

**In scope**

- The engine: parsers, the rule engine, TCP reassembly, the decoders
- The eBPF programs and the TC datapath
- The control socket and `cyphornctl`
- The diagnostic API
- Configuration handling, the installer package, and rule and intelligence
  updates
- **Privilege**: the engine runs as root and holds CAP_NET_ADMIN, CAP_NET_RAW,
  CAP_BPF and CAP_PERFMON. Anything that turns inspected traffic into
  influence over that process is in scope and is treated as high severity.

**Denial of service is in scope**, with one exception. An inline IPS that can
be switched off by the traffic it inspects has failed at its purpose, so:

- **In scope**: traffic that causes the engine to stop inspecting, stop
  forwarding, exhaust memory, or consume enough CPU to lose packets —
  including pathological input to a parser, a rule pattern, or reassembly.
  The engine bounds these deliberately; a way past a bound is a finding.
- **Out of scope**: overwhelming the hardware with volume alone. A 10 Gbps
  flood against a 1 Gbps appliance is arithmetic, not a vulnerability.

**Out of scope**

- Third-party dependencies — report those upstream; tell us too, and we will
  track the exposure
- Findings that require physical access to the appliance, or credentials the
  attacker should not have had
- Missing hardening that has no demonstrated impact, absent a path to one

## Recognition

Researchers are credited by name or handle in the advisory and the release
notes, unless anonymity is requested.
