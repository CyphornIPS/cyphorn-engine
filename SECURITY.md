# Security Policy — CyphornIPS

## Reporting a Vulnerability

If you discover a security vulnerability in CyphornIPS, report it
privately before disclosing it publicly. Do not open a public GitHub
Issue for security vulnerabilities.

**Contact:** cyphorntech@gmail.com

Include in your report:

- A description of the vulnerability and its potential impact.
- Steps to reproduce, or a proof-of-concept if available.
- The version of CyphornIPS affected (from the release tag or package
  version).
- Your assessment of severity, if you have one.

You will receive an acknowledgment within 5 business days, and a status
update at least every 14 days until the issue is resolved or declined.

## Safe Harbor for Good-Faith Security Research

CyphornIPS License (LICENSE.md) prohibits reverse engineering,
decompilation, and disassembly of the Software. This prohibition is
not intended to apply to good-faith security research conducted under
the terms below.

If you:

1. Make a good-faith effort to avoid privacy violations, data
   destruction, and service interruption to systems you do not own or
   do not have explicit authorization to test;
2. Only test against your own installation of CyphornIPS, in an
   isolated lab environment you control;
3. Report any vulnerability found privately to CyphornIPS through the
   contact above, before any public disclosure;
4. Do not exfiltrate, retain, or publish proprietary code, algorithms,
   or detection logic recovered during your research beyond what is
   strictly necessary to describe and reproduce the vulnerability in
   your report; and
5. Give CyphornIPS reasonable time to remediate before any public
   disclosure (a minimum of 90 days from acknowledgment, unless
   otherwise agreed in writing);

then CyphornIPS will not pursue legal action against you for the
specific reverse-engineering restriction in LICENSE.md Section 3(a),
solely to the extent necessary to conduct and report that research.
This safe harbor does not extend to redistribution, resale, commercial
use without permission, or any other restriction in LICENSE.md.

This safe harbor applies only to research conducted in accordance with
this policy. It does not apply to actions taken against systems or
networks you do not own or are not authorized to test, which remain
subject to applicable law regardless of this policy.

## Coordinated Disclosure Timeline

- Day 0: Report received, acknowledged within 5 business days.
- Day 0-90: CyphornIPS investigates and develops a fix. Status updates
  at least every 14 days.
- Day 90 (or earlier, by mutual agreement): Fix released. Public
  disclosure may follow, coordinated where possible so the fix is
  available before technical details are published.

If a fix cannot reasonably be completed within 90 days, CyphornIPS will
communicate a revised timeline and the reason for the delay.

## Scope

In scope:

- The CyphornIPS engine, eBPF/XDP components, and the installer package
  as distributed in official releases.
- The rules/intelligence update mechanism.

Out of scope:

- Third-party dependencies with their own disclosure process (report to
  the upstream project as well as to CyphornIPS if the issue is
  exploitable through CyphornIPS specifically).
- Denial-of-service findings that require unrealistic attacker
  privilege or physical access, unless clearly practical in a real
  deployment.

## Recognition

With your permission, CyphornIPS will credit you by name or handle in
the release notes for the fix, unless you request to remain anonymous.
