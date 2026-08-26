# CyphornIPS Update System & Provider Architecture

This document details the complete end-to-end lifecycle of the **CyphornIPS Update Engine**, including provider resolution, staging, SHA-256 integrity checks, native pre-validation, atomic installation, zero-downtime hot reload, and automated isolated rollback.

---

## 1. End-to-End Update Lifecycle

CyphornIPS executes updates via an isolated, multi-stage pipeline designed to ensure that **corrupted or syntactically invalid updates can never impact production traffic**.

```text
               ┌──────────────────────────────────────────┐
               │    GitHub / Generic HTTP Provider        │
               └────────────────────┬─────────────────────┘
                                    │ 1. Download manifest.json
                                    ▼
               ┌──────────────────────────────────────────┐
               │   Version Comparison (Local vs Remote)   │
               └────────────────────┬─────────────────────┘
                                    │ 2. Artifact Download
                                    ▼
               ┌──────────────────────────────────────────┐
               │     Download Artifact to /staging        │
               └────────────────────┬─────────────────────┘
                                    │ 3. Compute SHA-256
                                    ▼
                       [ SHA-256 Checksum Match? ]
                              /           \
                       NO    /             \    YES
                            ▼               ▼
                 [ REJECT UPDATE ]   ┌────────────────────────────────┐
                 [ 0 Files Mod.  ]   │  Stage Candidate in Sandbox    │
                                     │  (Candidate Managed + Local)   │
                                     └──────────────┬─────────────────┘
                                                    │ 4. Native In-Memory Parser
                                                    ▼
                                       [ Native Pre-Validation Pass? ]
                                              /           \
                                       NO    /             \    YES
                                            ▼               ▼
                                 [ REJECT UPDATE ]   ┌────────────────────────────────┐
                                 [ 0 Files Mod.  ]   │ Backup /managed/ snapshot      │
                                                     │ Atomically install new managed │
                                                     │ Update ownership.json          │
                                                     └──────────────┬─────────────────┘
                                                                    │ 5. Control Socket
                                                                    ▼
                                                     ┌────────────────────────────────┐
                                                     │ Live Zero-Downtime Hot Reload  │
                                                     │ (cyphorn_reload_rules)         │
                                                     └──────────────┬─────────────────┘
                                                                    │
                                                       [ Engine Reload Succeeded? ]
                                                              /           \
                                                       NO    /             \    YES
                                                            ▼               ▼
                                                 ┌────────────────┐   ┌───────────────┐
                                                 │ AUTO-ROLLBACK  │   │ UPDATE COMMIT │
                                                 │ Restore backup │   │ Gen++         │
                                                 │ Reload backup  │   │ version.json  │
                                                 │ Local untouched│   │ SUCCESS       │
                                                 └────────────────┘   └───────────────┘
```

---

## 2. Update Stages Explained

### Step 1: Provider Selection & Manifest Retrieval
The updater retrieves `manifest.json` from the configured provider (GitHub or generic HTTP):
```json
{
  "schema_version": 1,
  "product": "CyphornIPS",
  "channel": "stable",
  "rules": {
    "version": "2026.08.25.1",
    "filename": "rules.rules",
    "download_url": "https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/rules.rules",
    "sha256": "cbd01a33a50ba49d1fe0be6cb02cb996f41e8e195eec24818fa0cdf29dd294d6",
    "rule_count": 3204
  },
  "file_intelligence": {
    "version": "2026.08.25.1",
    "filename": "cyphorn-file-intelligence.json",
    "download_url": "https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/cyphorn-file-intelligence.json",
    "sha256": "4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a",
    "record_count": 1500
  }
}
```

### Step 2: Version Comparison Gate
The updater compares remote versions against local state in `/var/lib/cyphornips/version.json`:
- If `remote_version <= local_version` and `--force` is **not** set, the update exits with `All components are already up to date`.
- If `remote_version > local_version` or `--force` is active, the update proceeds.

### Step 3: Staging & SHA-256 Integrity Verification
- Artifacts are downloaded into a temporary staging area (`/var/lib/cyphornips/updates/staging/`).
- The engine computes the cryptographic SHA-256 hash of the downloaded payload via OpenSSL `EVP_sha256()`.
- **Integrity Gate**: If the computed hash does not match `manifest.rules.sha256`, the update is aborted immediately. Zero production files are touched.

> [!NOTE]
> In the current release, SHA-256 verifies transmission integrity against corruption and tampering. It serves as an integrity mechanism and is designed to be complemented by asymmetric cryptographic signatures (e.g. Minisign / Ed25519) in future releases.

### Step 4: Combined Sandboxed Pre-Validation
Before any active files on disk are touched, CyphornIPS constructs a complete temporary simulation environment:
```
/var/lib/cyphornips/updates/staging/combined_val_<pid>/
├── managed/
│   └── rules.rules          # Downloaded candidate rules
└── local/
    └── *.rules              # Exact copy of active local rules from /etc/cyphornips/rules/local/
```
The native engine loader (`rule_loader_load_directory()`) parses the combined tree. If even a single syntax error exists in the candidate rules:
- Pre-validation fails.
- The update is aborted with a descriptive parse error message.
- Active rules remain completely untouched.

### Step 5: Atomic Installation to `managed/` Only
- A backup snapshot of the current `/etc/cyphornips/rules/managed/` is taken in `/var/lib/cyphornips/updates/backup/`.
- Candidate rules are installed to `/etc/cyphornips/rules/managed/rules.rules`.
- Canonical rule fingerprints are computed and reconciled into `/var/lib/cyphornips/rules/ownership.json`.
- **`/etc/cyphornips/rules/local/` is NEVER touched or modified.**

### Step 6: Zero-Downtime Hot Reload & Isolated Rollback
- The updater issues a `reload rules` request over `/run/cyphornips/control.sock`.
- The live engine builds the new `RuleStore` in memory, compiles the fast Aho-Corasick trie, syncs eBPF map policies, and performs an atomic pointer swap.
- `rules_generation`, `detection_generation`, and `enforcement_generation` advance atomically.
- **Rollback on Reload Failure**: If the running engine rejects the reload, the updater automatically restores `/etc/cyphornips/rules/managed/` and `/var/lib/cyphornips/rules/ownership.json` from the backup snapshot, requests a reload of the restored backup, and leaves `local/` 100% intact.

---

## 3. Provider Abstraction Architecture

The update system is completely decoupled from any single hosting provider via the `CyphornUpdateProvider` interface (`include/update.h`):

```c
typedef struct CyphornUpdateProvider
{
    CyphornProviderType type;
    char provider_name[64];
    char manifest_url[512];
    void *provider_ctx;

    int (*fetch_manifest)(...);
    int (*fetch_artifact)(...);
    void (*cleanup)(...);
} CyphornUpdateProvider;
```

### Supported Providers:

### 3.1 GitHub Provider (`CYPHORN_PROVIDER_GITHUB`)
- **Default Provider**: Resolves manifests via GitHub Raw CDN.
- **Endpoint Structure**:
  `https://raw.githubusercontent.com/<repo>/<branch>/channels/<channel>/manifest.json`
- **Default Repository**: `CyphornIPS/cyphorn-rules`
- **Default Branch**: `main`
- **Default Channel**: `stable`
- **CLI Customization**:
  ```bash
  cyphornctl update check --repo custom-org/my-rules --branch dev --channel stable
  ```

### 3.2 Generic HTTP Provider (`CYPHORN_PROVIDER_GENERIC_HTTP`)
- Allows connecting CyphornIPS to arbitrary HTTP/HTTPS web servers, private corporate mirrors, S3 buckets, or Cloudflare R2 storage without channel dependencies.
- **CLI Customization**:
  ```bash
  cyphornctl update check --manifest-url https://updates.internal.lan/security/manifest.json
  ```

---

## 4. Update Channels

CyphornIPS officially distributes updates via the `stable` release channel:

| Channel | Purpose | Recommended Environment |
|---|---|---|
| `stable` (default) | Fully validated, regression-tested rule packages and verified threat intelligence. | Production enterprise firewalls, edge gateways. |

```bash
# Query updates on stable channel (default)
cyphornctl update check --channel stable

# Atomic update on stable channel
cyphornctl update all --channel stable
```

---

## 5. All-or-Nothing Atomic Semantics (`cyphorn_update_all`)

The `cyphornctl update all` command updates both Detection Rules and File Intelligence simultaneously with strict all-or-nothing guarantees:

1. Downloads Rules and File Intelligence artifacts to staging.
2. Verifies SHA-256 checksums for both.
3. Pre-validates both using native parsers.
4. **If either component fails validation, neither component is installed.**
5. If both pass, installs both and executes an atomic combined hot reload (`reload all`).
6. If the live reload fails, both components are rolled back to their respective previous backup versions simultaneously.
