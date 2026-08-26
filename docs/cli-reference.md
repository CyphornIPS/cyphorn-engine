# CyphornIPS CLI Reference (`cyphornctl`)

`cyphornctl` is the official administrative command-line control and update utility for **CyphornIPS**. It communicates with the running engine via a local Unix domain socket (`/run/cyphornips/control.sock`) and orchestrates rule/intelligence updates.

---

## 1. Global CLI Options

| Option | Short | Description | Example |
|---|:---:|---|---|
| `--socket <path>` | `-s` | Override default Unix control socket path (default: `/run/cyphornips/control.sock`). | `cyphornctl -s /tmp/my_control.sock status` |
| `--channel <name>` | | Set update channel (currently: `stable`, default: `stable`). | `cyphornctl update check --channel stable` |
| `--manifest-url <url>` | | Use generic HTTP provider pointing directly to a manifest URL. | `cyphornctl update check --manifest-url https://mirror.lan/manifest.json` |
| `--repo <owner/repo>` | | Override GitHub repository (default: `CyphornIPS/cyphorn-rules`). | `cyphornctl update check --repo myorg/cyphorn-rules` |
| `--branch <branch>` | | Override GitHub branch (default: `main`). | `cyphornctl update check --branch staging` |
| `--force` | | Force update/re-installation even if already at latest version. | `cyphornctl update rules --force` |
| `--json` | | Output responses in structured JSON format. | `cyphornctl status --json` |
| `--help` | `-h` | Display usage and help message. | `cyphornctl --help` |
| `--version` | | Display CLI version information. | `cyphornctl --version` |

---

## 2. Update Subcommands

### 2.1 `cyphornctl update check`
- **Syntax**: `cyphornctl update check [options]`
- **Purpose**: Queries the remote update provider, compares remote versions against local state, and reports whether updates are available.
- **Side Effects**: **Zero.** Does not download artifacts or modify any local files.
- **Example**:
  ```bash
  cyphornctl update check
  ```
- **Expected Output**:
  ```text
  ================================================================================
                    CyphornIPS Update Availability Diagnostics                    
  ================================================================================
    Provider           : GitHub Update Provider
    Channel            : stable
    Manifest URL       : https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/manifest.json

  ----------------------------- Detection Rules ---------------------------------
    Installed Version  : 2026.08.24.1 (1 rules)
    Available Version  : 2026.08.25.1 (3204 rules)
    Status             : [UPDATE AVAILABLE]
    Download URL       : https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/rules.rules
    Expected SHA-256   : cbd01a33a50ba49d1fe0be6cb02cb996f41e8e195eec24818fa0cdf29dd294d6

  --------------------------- File Intelligence ---------------------------------
    Installed Version  : 2026.08.24.1 (1 records)
    Available Version  : 2026.08.25.1 (1500 records)
    Status             : [UPDATE AVAILABLE]
    Download URL       : https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/cyphorn-file-intelligence.json
    Expected SHA-256   : 4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a
  ================================================================================
  ```

---

### 2.2 `cyphornctl update migrate`
- **Syntax**: `cyphornctl update migrate`
- **Purpose**: Migrates a legacy flat rules directory (`/etc/cyphornips/rules/*.rules`) into the authoritative hierarchical structure (`/etc/cyphornips/rules/managed/` and `/etc/cyphornips/rules/local/`).
- **Safety**: Computes canonical fingerprints for all rules, pre-validates in sandbox, and verifies engine reload before committing. Zero rule loss.
- **Example**:
  ```bash
  cyphornctl update migrate
  ```
- **Expected Output**:
  ```text
  OK: Successfully migrated rules to hierarchical managed/ and local/ structure (3205 active rules loaded).
  ```

---

### 2.3 `cyphornctl update rules`
- **Syntax**: `cyphornctl update rules [--force] [options]`
- **Purpose**: Downloads candidate rules, validates integrity via SHA-256, pre-validates syntax combined with active local rules in sandbox, atomically installs to `/etc/cyphornips/rules/managed/rules.rules`, and executes zero-downtime hot reload.
- **Protection**: Does **not** modify or delete any file in `/etc/cyphornips/rules/local/`.
- **Example**:
  ```bash
  cyphornctl update rules
  ```
- **Expected Output**:
  ```text
  OK: Rules updated successfully to version 2026.08.25.1 (3204 rules loaded). Engine reload confirmed.
  ```

---

### 2.4 `cyphornctl update intelligence`
- **Syntax**: `cyphornctl update intelligence [--force] [options]`
- **Purpose**: Downloads candidate File Intelligence threat dataset, verifies SHA-256, pre-validates JSON schema, atomically installs to `/etc/cyphornips/rules/cyphorn-file-intelligence.json`, and executes zero-downtime hot reload.
- **Example**:
  ```bash
  cyphornctl update intelligence
  ```
- **Expected Output**:
  ```text
  OK: File Intelligence updated successfully to version 2026.08.25.1 (1500 records loaded). Engine reload confirmed.
  ```

---

### 2.5 `cyphornctl update all`
- **Syntax**: `cyphornctl update all [--force] [options]`
- **Purpose**: Atomically downloads, verifies, pre-validates, installs, and hot-reloads both Detection Rules and File Intelligence simultaneously with strict all-or-nothing rollback semantics.
- **Example**:
  ```bash
  cyphornctl update all
  ```
- **Expected Output**:
  ```text
  OK: Successfully updated both Rules (v2026.08.25.1, 3204 rules) and File Intelligence (v2026.08.25.1, 1500 records). Engine reload confirmed.
  ```

---

## 3. Control & Hot Reload Subcommands

### 3.1 `cyphornctl reload rules`
- **Syntax**: `cyphornctl reload rules`
- **Purpose**: Instructs the running engine to reload all detection rules from `/etc/cyphornips/rules/` without restarting the process.
- **Expected Output**:
  ```text
  OK: Reloaded 3204 rules successfully (0 errors, generation 5)
  ```

---

### 3.2 `cyphornctl reload intelligence`
- **Syntax**: `cyphornctl reload intelligence`
- **Purpose**: Instructs the running engine to reload File Intelligence threat dataset from `/etc/cyphornips/rules/cyphorn-file-intelligence.json` without restart.
- **Expected Output**:
  ```text
  OK: Reloaded 1500 File Intelligence records successfully (generation 4)
  ```

---

### 3.3 `cyphornctl reload all`
- **Syntax**: `cyphornctl reload all`
- **Purpose**: Atomically reloads both Detection Rules and File Intelligence simultaneously.
- **Expected Output**:
  ```text
  OK: Reloaded all subsystems successfully (Rules: 3204, File Intelligence: 1500)
  ```

---

### 3.4 `cyphornctl status` & `cyphornctl status json`
- **Syntax**: `cyphornctl status` or `cyphornctl status json` (or `cyphornctl status --json`)
- **Purpose**: Queries live runtime telemetry, PID, uptime, generations, reload counts, and version state.
- **Expected Output**: See [docs/telemetry.md](file:///root/cyphorn-engine/docs/telemetry.md).

---

### 3.5 `cyphornctl ping`
- **Syntax**: `cyphornctl ping`
- **Purpose**: Verifies IPC connectivity to the running CyphornIPS engine control socket.
- **Expected Output**:
  ```text
  PONG
  ```
- **Exit Code**: `0` if engine is reachable, `1` if unreachable.
