# CyphornIPS Troubleshooting Guide

This guide documents real-world operational issues, error messages, failure modes, diagnostic procedures, and corrective actions encountered during administration and development of **CyphornIPS**.

---

## Issue 1: `cyphornctl: invalid option or unknown command 'update'`

### Symptom
Running `cyphornctl update check` or `cyphornctl update all` outputs:
```text
Error: Unknown command 'update'. Supported commands: reload rules, reload intelligence, reload all, status, status json, ping
```

### Cause
The system has an older version of the `cyphornctl` CLI installed in `/usr/local/bin/cyphornctl` or `/usr/bin/cyphornctl` that predates the Update Engine implementation.

### Diagnosis Command
```bash
which cyphornctl
cyphornctl --version
```

### Corrective Action
Recompile and reinstall the latest `cyphornctl` binary from the repository:
```bash
cd /root/cyphorn-engine
make clean && make -j$(nproc)
sudo cp bin/cyphornctl /usr/local/bin/cyphornctl
sudo cp bin/cyphorn-engine /usr/local/bin/cyphorn-engine
```

---

## Issue 2: GitHub 404 Manifest Not Found

### Symptom
Running `cyphornctl update check` fails with:
```text
Error: Failed to fetch manifest from 'https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/manifest.json': HTTP status 404 (Not Found)
```

### Cause
1. The remote GitHub repository, branch, or channel path does not exist yet.
2. An incorrect repository owner/name or branch was supplied.

### Diagnosis Command
```bash
curl -I https://raw.githubusercontent.com/CyphornIPS/cyphorn-rules/main/channels/stable/manifest.json
```

### Corrective Action
- If using a private mirror or custom fork, specify the repository explicitly:
  ```bash
  cyphornctl update check --repo YourOrg/your-cyphorn-rules --branch main
  ```
- Or use the Generic HTTP Provider pointing directly to your manifest:
  ```bash
  cyphornctl update check --manifest-url https://your-server.lan/channels/stable/manifest.json
  ```

---

## Issue 3: Engine Not Running During Update (Offline Mode)

### Symptom
Running `cyphornctl update all` outputs:
```text
Notice: Files installed to disk, but CyphornIPS engine is not running on /run/cyphornips/control.sock.
Hot reload was skipped. Changes will take effect upon engine startup.
```

### Cause
The updater successfully installed the validated artifacts to disk, but the `cyphorn-engine` process is not actively running, or its control socket is located on a non-standard path.

### Diagnosis Command
```bash
cyphornctl ping
ps aux | grep cyphorn-engine
```

### Corrective Action
Start or restart the engine daemon:
```bash
sudo systemctl start cyphornips
# Or interactive mode:
sudo cyphorn-engine -w enp0s3 -l enp0s8
```

---

## Issue 4: SHA-256 Checksum Mismatch Protection

### Symptom
Update fails immediately before staging or modifying files:
```text
Error: Rules SHA256 checksum mismatch! Expected 'cbd01a33a50ba49d1fe0be6cb02cb996f41e8e195eec24818fa0cdf29dd294d6', got '1111222233334444555566667777888899990000aaaabbbbccccddddeeeeffff'. Update aborted; active configuration untouched.
```

### Cause
The downloaded artifact was corrupted during transit, or the upstream manifest's `sha256` field is out of sync with the actual uploaded artifact file.

### Diagnosis Command
```bash
# Check the SHA-256 of the downloaded artifact manually:
sha256sum /var/lib/cyphornips/updates/staging/rules.rules
```

### Corrective Action
1. Re-run with `--force` to retry the download:
   ```bash
   cyphornctl update rules --force
   ```
2. Verify upstream repository release metadata and regenerate `manifest.json`.

---

## Issue 5: Syntax Validation Gate Rejection

### Symptom
The update downloads and passes SHA-256, but is rejected before installation:
```text
CyphornIPS: invalid rule at /var/lib/cyphornips/updates/staging/combined_val_1829/managed/rules.rules:142
Error: Rules validation failed with 1 parse errors. Active configuration unchanged.
```

### Cause
The candidate rules file contains invalid Snort syntax (e.g. malformed option, unknown protocol, invalid regex, or unclosed parenthesis).

### Diagnosis Command
Inspect the failing line reported in the error message:
```bash
sed -n '140,145p' /var/lib/cyphornips/updates/staging/rules.rules
```

### Corrective Action
Because CyphornIPS rejected the update at the pre-validation gate, **active engine rules were not modified and zero packets were affected**. Fix the syntax error in the upstream ruleset before republishing.

---

## Issue 6: Automatic Rollback After Engine Reload Failure

### Symptom
The update installs files, but engine reload encounters an error, triggering automatic rollback:
```text
CyphornIPS: reload failed: rule memory allocation error
Notice: Engine reload failed. Automatically rolled back managed rules to previous version (v2026.08.24.1).
Engine remains operational on previous configuration.
```

### Cause
The candidate rules passed initial parsing, but the running engine could not allocate sufficient memory for the compiled Aho-Corasick state machine or eBPF policy maps.

### Diagnosis Command
```bash
cyphornctl status
cat /var/log/cyphornips/cyphornips.log | tail -n 20
```

### Corrective Action
- The updater automatically restored the previous backup snapshot from `/var/lib/cyphornips/updates/backup/`.
- Local user rules in `/etc/cyphornips/rules/local/` remained 100% untouched throughout the rollback.
- Check engine memory limits in `/etc/cyphornips/cyphornips.conf`.

---

## Issue 7: Duplicate Rules Ballooning Rule Count (Flat vs. Hierarchical)

### Symptom
After running an update or reloading, `Loaded Rules` in `cyphornctl status` doubles (e.g. from 3204 to 6408).

### Cause
An upstream `rules.rules` file was placed directly in a flat `/etc/cyphornips/rules/` directory alongside existing individual `.rules` files without running the migration tool. The engine loads all `.rules` files in the directory recursively.

### Diagnosis Command
```bash
ls -la /etc/cyphornips/rules/
```

### Corrective Action
Run the migration subcommand to split upstream rules from local rules:
```bash
cyphornctl update migrate
```
After migration:
- Upstream rules are consolidated in `/etc/cyphornips/rules/managed/rules.rules`.
- Local custom rules are preserved in `/etc/cyphornips/rules/local/`.

---

## Issue 8: Difference Between `version.json` and Live Runtime Telemetry

### Symptom
`/var/lib/cyphornips/version.json` shows version `2026.08.25.1`, but the UI or `cyphornctl status` shows an older generation or different rule count.

### Cause
`version.json` reflects the on-disk installed version state. If the engine was reloaded with local manual edits or if the engine was restarted without loading the updated configuration, the runtime state may differ from on-disk version state.

### Diagnosis Command
```bash
# Compare on-disk version against live engine telemetry:
cat /var/lib/cyphornips/version.json
cyphornctl status
```

### Corrective Action
Force an atomic reload to synchronize the live engine with the on-disk state:
```bash
cyphornctl reload all
```

---

## Issue 9: `Loaded Rules` Greater than Manifest `rule_count`

### Symptom
The upstream manifest reports `rule_count: 3204`, but `cyphornctl status` reports `Loaded Rules: 3207`.

### Cause
This is **expected and normal**. `Loaded Rules` represents the **total active rules in memory**, which is the sum of:
$$\text{Loaded Rules} = \text{Managed Rules (from Upstream)} + \text{Local Rules (from /etc/cyphornips/rules/local/)}$$

If you have 3 local rules in `/etc/cyphornips/rules/local/custom.rules` and 3204 managed rules in `/etc/cyphornips/rules/managed/rules.rules`, the engine will load exactly $3204 + 3 = 3207$ rules.
