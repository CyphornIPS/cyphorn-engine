# 🛠️ CyphornIPS — Daily Operations

**The short one.** Everything a running appliance needs from you, day to day.
The [full manual](https://cyphornips.github.io/cyphorn-engine/) is the reference;
this page is the routine.

---

## ⏱️ The morning check

Four commands. If all four are clean, the appliance is inspecting, enforcing and
accounting for itself.

```bash
systemctl is-active cyphornips      # active
cyphornctl status                   # health, coverage, per-link loss
cyphornctl logs threats -n 20       # what was blocked or flagged
cyphornctl diagnose                 # anything stopping it doing its job
```

### What to read in `status`

| Line | Healthy | What anything else means |
|---|---|---|
| `Health` | `OK` | `DEGRADED` / `FAILED` names its own reason; `diagnose` prints the fix |
| `Coverage` | invariant holds | an unexplained residual — frames that resolved to no state. **Never ignore this line.** |
| `Inspected Links` | every link `lost=0 0%` | sustained loss on one link means its capture ring is smaller than its load |
| `Rules` | the count you expect | a lower count after a reload means rules were **rejected** |
| `Enforcement` | `enforcing` | `degraded` means verdicts are made but not fully applied |

> [!IMPORTANT]
> `Coverage` is the line that makes CyphornIPS different from a dashboard that
> always looks green. Every frame must resolve to exactly one state — inspected
> and allowed, inspected and dropped, or uninspected **with a named reason**. A
> residual means something went past unaccounted for.

---

## 📋 Routine tasks

### Change rules or policy

```bash
sudo nano /etc/cyphornips/domain_policy.conf   # or country-policy.conf, or a rule file
cyphornctl reload all                          # no packet is dropped by a reload
cyphornctl status                              # confirm the rule count went UP
cyphornctl logs rules -n 40                    # any rule refused, and why
```

> [!WARNING]
> **Count the rules after every reload.** A rule that does not parse does not
> stop the reload — it is refused and the rest continue. If the count goes
> *down*, you have lost detection while believing you added some.

### Someone says a site is blocked

```bash
cyphornctl explain 192.168.1.50 example.com            # the decision for that client
cyphornctl logs threats --blocked -n 40                # what was enforced recently
grep -i example.com /etc/cyphornips/rules/feeds/*.lst  # is a feed the reason?
```

`explain` answers for **that one client**, not the network in general — which
matters behind NAT, where devices share one address on the outside interface.

### Watch it live

```bash
cyphornctl logs threats --blocked -f   # enforcement as it happens
cyphornctl logs alerts -f              # every alert, including policy allows
cyphornctl logs engine -f              # start-up, reloads, audit records
cyphornips                             # full-screen terminal dashboard
```

### Update rules and intelligence

```bash
cyphornctl update check          # what is available; changes nothing
cyphornctl update all            # download, verify, install, hot reload
cyphornctl status                # confirm the new generation is live
```

Every update is verified twice before it installs: **Ed25519** on the manifest,
**SHA-256** on each artifact. Nothing installs if either fails, and there is no
phone-home.

### Disk and logs

```bash
du -sh /var/log/cyphornips/ /var/lib/cyphornips/
cyphornctl config | grep -E 'log_(max|rotate|level)'
journalctl -u cyphornips --since today | tail -40
```

---

## 🚨 When something is wrong

**Always start here.** It reads the configuration and the system directly, works
**while the engine is down**, and every finding carries the command that fixes it.

```bash
cyphornctl diagnose
```

Then, in order:

```bash
systemctl status cyphornips
journalctl -u cyphornips -n 80 --no-pager
cyphornctl config                 # the settings in force, not the file's text
for i in <wan> <lan>; do echo "$i $(cat /proc/sys/net/ipv4/conf/$i/forwarding)"; done
```

### Common situations

| Symptom | First thing to check |
|---|---|
| Nothing behind the appliance routes | forwarding on each inspected link — see the `fail_mode` note below |
| Rule count dropped after a reload | `cyphornctl logs rules` — a refused rule names its own reason |
| `Health: DEGRADED` | `cyphornctl status` prints the reason on the same line |
| Loss on one link only | that link's ring is smaller than its load; ring sizing is per host |
| A country rule does nothing | `cyphornctl diagnose` — GeoIP database present? |
| Legitimate site blocked | `cyphornctl explain <client-ip> <domain>` |

> [!CAUTION]
> **`fail_mode = closed` means stopping the service takes transit down** on every
> inspected link — by design. To hand routing back by hand while the engine is
> stopped:
> ```bash
> sudo cyphorn-engine --restore-fail-mode --force
> ```
> Know this **before** your first `systemctl stop`, not after.

---

## 📁 The files you will open

| Path | What it is |
|---|---|
| `/etc/cyphornips/cyphornips.conf` | main configuration — interfaces, workers, fail mode |
| `/etc/cyphornips/domain_policy.conf` | domain blocking by category |
| `/etc/cyphornips/country-policy.conf` | allow and drop by country and direction |
| `/etc/cyphornips/rules/local/` | **your** rules — never touched by an update |
| `/etc/cyphornips/rules/feeds/` | threat feeds, rebuilt daily by the timer |
| `/var/log/cyphornips/alerts.log` | every alert |
| `/var/log/cyphornips/rules.log` | rule matches, and why a rule was refused |
| `/var/log/cyphornips/cyphornips.log` | the engine's operational log |

You do not need to memorise any of these. `cyphornctl logs <name>` asks the
engine where it is writing and reads from there, and `cyphornctl config` prints
the settings actually in force — a key you set that is missing from that output
was not understood.

---

## ⌨️ Command cheat sheet

| Command | What it does |
|---|---|
| `cyphornctl status` | health, coverage, rules, per-link loss |
| `cyphornctl status json` | the same, for a collector (`--compact` for one line) |
| `cyphornctl diagnose` | what is stopping it, with the fix — **works while down** |
| `cyphornctl config` | the configuration in force |
| `cyphornctl logs threats` | blocked or flagged only, in columns, repeats folded |
| `cyphornctl logs threats --blocked` | narrowed to what was actually enforced |
| `cyphornctl logs alerts` \| `rules` \| `engine` | the three logs, by name not by path |
| `cyphornctl explain <ip> <domain>` | why this client was allowed or blocked |
| `cyphornctl reload rules` \| `all` | hot reload, no packet dropped |
| `cyphornctl update check` \| `all` | signed rule and intelligence updates |
| `cyphornctl ping` | is the engine listening? |
| `cyphornips` | full-screen terminal dashboard |

Add `-n <N>` for how many lines (with `threats`, how many *threats*), and `-f` to
follow. `cyphornctl --help` prints all of it.

---

## 🔒 Weekly, and after every change

- [ ] `cyphornctl diagnose` reports nothing blocking
- [ ] `Coverage` invariant holds
- [ ] Rule count is what you expect
- [ ] `cyphornctl logs threats -n 50` — anything surprising in what was blocked?
- [ ] Disk under `/var/log/cyphornips` is not growing without bound
- [ ] Feeds refreshed: `systemctl status cyphornips-feeds.timer`
- [ ] Your configuration backup is current:
      `sudo tar czf /root/cyphornips-config-$(date +%F).tar.gz /etc/cyphornips`

> [!NOTE]
> **CyphornIPS is one layer.** Run it alongside a default-deny firewall, endpoint
> protection, patching, monitored administrative access, off-box log retention,
> and backups you have actually restored from. No IPS detects everything, and
> this one says so in [its own limitations](https://cyphornips.github.io/cyphorn-engine/#limitations).

---

📘 **Full manual:** <https://cyphornips.github.io/cyphorn-engine/>
📧 **info@cyphorn.com**
