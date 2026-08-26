# CyphornIPS Official Documentation Index

Welcome to the **CyphornIPS** official technical documentation suite. This documentation covers all architectural concepts, administrative operations, rule writing, update management, and troubleshooting.

---

## Documentation Modules

### 1. [CLI Reference (`docs/cli-reference.md`)](file:///root/cyphorn-engine/docs/cli-reference.md)
Complete command-line reference for `cyphornctl`, including options (`-s`, `--channel`, `--manifest-url`, `--repo`, `--branch`, `--force`, `--json`), update commands (`check`, `migrate`, `rules`, `intelligence`, `all`), hot reload commands, status diagnostics, and exit codes.

### 2. [Rule Engine & Fingerprinting (`docs/rules.md`)](file:///root/cyphorn-engine/docs/rules.md)
Snort-compatible rule syntax, sticky DPI inspection buffers (`http.uri`, `tls.sni`, `dns.query`, `file.data`), Managed vs. Local rules separation (`/etc/cyphornips/rules/managed/` and `/etc/cyphornips/rules/local/`), and Canonical Rule Fingerprinting (deterministic SHA-256 identity independent of SID).

### 3. [Rule Action Precedence (`docs/action-precedence.md`)](file:///root/cyphorn-engine/docs/action-precedence.md)
Authoritative action precedence hierarchy ($\mathbf{PASS} > \mathbf{DROP} > \mathbf{ALERT}$), conflict resolution semantics, telemetry accounting, and why rule ownership does not alter action evaluation.

### 4. [Telemetry & UI Metrics (`docs/telemetry.md`)](file:///root/cyphorn-engine/docs/telemetry.md)
Operational status tags and abbreviations (`[IDLE]`, `[MATCHED]`, `[EFFECTIVE]`, `[OVERRIDDEN]` / `OVRD`, `[DROPPED]`, `[ALERTED]`), two-stage evaluation pipeline, interactive Ncurses Dashboard layout, Rule Details inspector, and JSON metrics.

### 5. [Update Engine & Providers (`docs/updates.md`)](file:///root/cyphorn-engine/docs/updates.md)
End-to-end update lifecycle, GitHub Provider Adapter, Generic HTTP Provider, release channels (`stable`), SHA-256 integrity verification, combined sandboxed pre-validation, atomic installation, zero-downtime hot reload, generation synchronization, and automated isolated rollback.

### 6. [Troubleshooting Guide (`docs/troubleshooting.md`)](file:///root/cyphorn-engine/docs/troubleshooting.md)
Diagnostic procedures and solutions for real-world issues: CLI path mismatches, GitHub 404 errors, offline engine updates, SHA-256 mismatches, syntax validation rejections, automatic rollback after reload failure, duplicate rule ballooning, and telemetry discrepancies.

### 7. [Developer & Systems Architecture (`docs/architecture.md`)](file:///root/cyphorn-engine/docs/architecture.md)
Internal subsystem data flow, eBPF packet capture, stateful flow reassembly, atomic generation model, Unix domain socket IPC protocol, and codebase organization.
