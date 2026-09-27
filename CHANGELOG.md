# Changelog

Notable changes per release. Dates are the day the work landed, not the day a
package was cut.

The entries say what changed and, where it matters, what it was before —
because the reason a setting exists is usually the thing you need when you are
deciding whether to touch it.

---

## 1.8.0 — 25 September 2026

> **1.5.0 is the release this follows.** The 1.7.x entries below were never
> published: they are development milestones from 22–24 September, kept
> because each one records a defect and how it was found, and because an
> operator reading them is reading the history of the release they are
> installing. Nobody ran 1.7.x, so every upgrade note here is written for
> somebody coming from **1.5.0**.


**The inspection threads run.** Under a 4096-flow flood on a four-core
appliance, three workers inspected every frame offered where one inspected
65% of them:

| | one worker | three workers |
|---|---|---|
| inspected | 799,662 of 1,216,552 — **65%** | 1,085,443 of 1,085,443 — **100%** |
| CPU spread | 55% / 0 / 0 | **28.6% / 13.3% / 13.6%** |

`worker_count = 0` in `cyphornips.conf` derives one thread per core less one,
which is three here. `cyphornctl config` reports the configured value and the
one in force.

### The steering program was never loaded

`bpf/cyphorn_fanout.bpf.o` was written, built, and unit-tested by
`tests/test_flow_fanout.c` — and **no line of the engine opened it**. Joining a
`PACKET_FANOUT_EBPF` group with no program makes the kernel hand every frame
to one socket, so three workers shared one busy thread while every surface
reported three running. It is loaded now, and shipped: without it on an
installed appliance the fallback is silent.

**No gate caught this, and that is the part worth keeping.** Each checked its
own half, and all of them passed on an engine where two threads of three were
idle — `flow-fanout` checked the program's logic but not whether it was
loaded; `worker-equivalence` checked that verdicts do not change but not that
anything was spread; `coverage` checked that the equation balances but not who
inspected what. `make worker-scaling` is the gate that asks whether the
threads carry load, and it floods many flows because a single flow is the one
case flow affinity cannot spread by design — measuring that first produced a
meaningless 51% against 52%.

### The same mistake, made twice

Per-worker counters were added and the per-packet context feeding them was
left shared. The first time it was `EngineState.packet`: the engine died on
its first traffic and the cause was obvious within the hour.

The second time it was `EngineState.pipeline` — the structure that carries one
packet through the stages and records where it ended — across 72 call sites.
**It did not crash.** It ran, and quietly produced terminal states short of
`engine_received`, which the coverage invariant reported on the live appliance
as `VIOLATED(-184)`.

The tell is worth keeping: measured at -184, -184, -191 across 301 new frames.
**A delta that stays flat while traffic climbs is a race on shared per-packet
state; a delta that grows with traffic is a leak.** Both invariants now read
`HOLDS(+0)` under three workers on a live appliance.

`make worker-packet-context` lists all five members rather than four. Adding
one to that list is what makes the next omission impossible.

### Two more the threads exposed

- **The sanitizer build stopped linking.** `cyphorn_worker_id()` was made a
  `static inline`, and a build without optimisation does not inline it, so the
  ThreadSanitizer target failed on an undefined reference. That is the one
  build that must not break during concurrency work, since it is the only one
  that can find a data race. It is a macro now.

- **`src/worker_pool.c` opened sockets the attack-surface inventory did not
  declare.** `make management-plane` named the file. Declared as
  `worker_capture`: one AF_PACKET set per inspection thread, joined to a
  per-interface fanout group, needing `CAP_NET_RAW` and `CAP_BPF`, on by
  default whenever `worker_count` resolves above one. An inventory that omits
  three threads' sockets is one nobody can review.

### And a cost this release introduced, then removed

Moving the per-packet decode context behind accessors put a call to
`cyphorn_worker_id()` — an external function in another translation unit — on
the hot path 575 times. Inlining it as a thread-local read took the
**one-worker** figure from 51% to 65%: a net loss of my own making rather than
a threading gain, and worth separating from the numbers above.

### Still not measured

`make performance` replays captures offline, and offline replay is forced to
one worker so its digest stays reproducible. Its p99 therefore cannot move
however good threading becomes — that gate is the wrong instrument for this
question, and a new one is needed for latency under concurrency.

---

## 1.7.3 — 24 September 2026  *(never published; folded into 1.8.0)*

The inspection threads, built and then switched off — and the measurement that
switched them off is the useful part of this entry.

### What was built, and works

- `src/inspect.c` — the packet path as its own translation unit. It was in
  main.c because the capture loop was, which is a layering accident: the
  linker found it when the worker pool needed it, since main.o is the one
  object `cyphornctl` does not link.
- `src/worker_pool.c` — each worker opens its own sockets on the same
  interfaces and joins a per-interface `PACKET_FANOUT` group.
- `packet_io_join_fanout()` — `PACKET_FANOUT_EBPF`, not `HASH`. HASH mixes
  `skb->hash`, computed from the tuple as it appears, so a client's packet and
  the server's reply land on different sockets and two workers each hold half
  a TCP stream. Neither `FLAG_DEFRAG` (the engine does its own, and counts it)
  nor `ROLLOVER` (which moves a flow between sockets — the one thing this
  design exists to prevent).
- The flow budget is **divided** among workers, inside
  `connection_table_capacity_for()`, after the zeros are resolved. A
  configuration that sets no explicit bound carries zeros and dividing zero
  divides nothing: each worker built a full-size table, three asked for three
  times the flow memory, and the engine died with "connection table
  initialization failed". Without this it does not start at all.

Three workers start, join their groups, and the table splits 4096 → 1365 each.

### And then it dies on the first traffic

| build | result |
|---|---|
| `-O2` | dies on the first packets |
| ASan | 600 frames, **zero findings** |

A build that passes its sanitizer and dies without it is a data race, and
nothing else looks like that: ASan serialises enough to change the
interleaving.

`EngineState` carries the packet being decoded and the verdict being formed as
single shared members — `packet`, `current_verdict`, `current_drop_reason`,
`current_response_done` — and the inspection path writes them **493 times**
across `decoder.c` and `inspect.c`, once per packet, per worker.

The flow tables, the trackers and the per-rule counters *are* per-worker, and
have been. `worker.h` named the capture loop as the one thing in the way; that
was true as far as it went, and it did not mention this. The loop is split and
this is what was underneath.

### So threading is off, and says why in the code

`threading_ready()` returns false, with the measurement written where the
decision is. `worker_count` above one is refused out loud, naming what was
asked for and what is running — a setting that is quietly ignored is
indistinguishable from one that worked.

`make worker-packet-context` is new and checks the two facts against each
other: it fails if threading is enabled while those members are shared, and it
fails if they move and nobody turns threading on. The estimate that this was
one step was wrong, and the gate is there so the next estimate is a reading.

### `worker_count`

```
worker_count = 0     # 0 = one per core, less one for everything else
```

Derived from the host, like the capture ring and the rule ceiling — a size,
not a behaviour branching on detected hardware. On a four-core appliance it
suggests three. `cyphornctl config` reports the configured value and the one
in force.

---

## 1.7.2 — 24 September 2026  *(never published; folded into 1.8.0)*

Two pieces of work: the threat intelligence, which turned out to be a
scheduling fault rather than a data one, and the capture loop, which is now
two functions instead of one.

### The feeds were never refreshed

`/usr/local/sbin/cyphorn-refresh-feeds.sh` shipped on 2026-09-15, worked, and
**nothing ever ran it.** No timer, no cron entry — the script carried a cron
line inside a comment, and a comment schedules nothing. Meanwhile
`cyphornips.conf` told the operator the lists were "refreshed daily".

One run recovered what nine days had cost:

| list | was | now |
|------|-----|-----|
| malware hostnames (URLhaus) | 176 | **3,615** |
| IP reputation | 2,913 | **5,187** |
| file hashes (MalwareBazaar) | **101**, dated 5 Sep | **1,960** |
| bad certificates | 10,715 | 10,793 |

`cyphornips-feeds.timer` — daily, `RandomizedDelaySec=2h`, `Persistent=true` —
makes the configuration file's claim true. Enabled from postinst.

**Sources added.** ThreatFox, as a new `c2_domains` feature: URLhaus lists
where malware was *distributed*, ThreatFox lists where it *calls home*, and
the second is the half still visible after the payload is already on a device.
3,143 C2 hostnames and 2,274 addresses, filtered at confidence >= 75 — on a
household uplink a false positive is a site somebody cannot reach.
MalwareBazaar now refreshes the hash dataset daily instead of once by hand.

**And a second defect in the same script:** it called `cyphornctl reload
rules`, but file intelligence is a separate subsystem with a separate reload.
The hashes were written and the engine went on matching the previous set —
the file held 1,960 while the status surface reported 101. Now `reload all`.

### The capture loop is two jobs, and now two functions

One 885-line function interleaved inspecting a packet with running the
appliance: reload requests, the control socket, health and the watchdog ping,
the forwarding re-assert, the coverage refresh. A worker thread cannot run
that — four workers must not each reload the rule set or redraw a screen — and
`include/worker.h` has named it as the one thing standing between this engine
and more than one inspection thread.

It is now `run_appliance_tick()` and `inspect_one_packet()`, and the loop
between them is 25 lines. Nothing else changed: the same statements, in the
same order, behind two names.

Two details worth recording. The packet half contained eight `continue` and
`break` statements, which mean nothing inside a function; they became a
returned verdict, and rewriting them mechanically was safe for a checkable
reason — **the block contains no `for`, `while` or `switch` at all**, so not
one of them could have belonged to an inner construct. And three dependencies
the compiler found that a grep had not: a per-packet timer that looked like
process state, a local `config` (`engine->config` is a copy of it, taken once
and never written again — checked), and an `&` that a blanket rewrite had
eaten.

### `worker_count`

```
worker_count = 0     # 0 = derive from this machine
```

Zero means one thread per core, less one for everything that is not
inspection. A size taken from the host, like the capture ring and the rule
ceiling — not a behaviour that branches on detected hardware.

**Threads are not started yet.** A value above one is refused out loud, naming
what it asked for and what it got, because a setting that is quietly ignored
is indistinguishable from one that worked. `cyphornctl config` reports both
the configured value and the one in force.

The refusal was first put inside `cyphorn_worker_set_count()`, and
`make unit-suites` caught it within the hour: `test_worker_state_isolation`
declares four workers on one thread, which is the only way the per-worker
arithmetic is checked at all, and clamping the count broke all 27 of its
checks. `worker.h` says why that separation exists. The refusal moved to where
threads would actually be started.

### Verified

71 of 72 gates pass, 0 fail. `performance` still exceeds its own
timeout and its p99 budget; that is the single-thread ceiling, and starting
the threads is the next piece of work.

---

## 1.7.1 — 23 September 2026  *(never published; folded into 1.8.0)*

Everything here was found in one day of running 1.7.0 on a live appliance, and
most of it was found because the appliance stopped carrying traffic. The theme
is the same in every entry: a number that was true once, reported as though it
were true now.

### Capture loss was drain starvation, not ring size

**Every inspected link past the busiest one was losing frames — up to half of
what it carried — and the totals made it look like a ring too small to hold a
burst.**

`packet_io_drain_rings()` scanned ports from zero and returned as soon as it
took one frame. A link busy enough to always have a frame waiting meant the
ports behind it were never drained at all: their rings filled and the kernel
dropped into them. With two links the bias was survivable. With four it was
not, which is why it surfaced with the multi-interface support of 1.7.0 and not
before.

The two faults produce the same total and need opposite fixes, so they are
worth telling apart: **a ring that is too small loses frames in proportion to a
link's rate; a starved drain loses them in proportion to the link's position.**
Measured on four links, a flood into port 0 and a ~200 frames/second trickle
into port 3:

| | before | after |
|---|---|---|
| port 0, flooded | 20% lost | 20% lost — a real capacity limit, not a defect |
| port 3, trickle | **48% lost** | **0% lost** |

Losing 48% of 200 frames a second is not something a 2 MiB ring can do at that
rate. The drain now resumes where it left off. `make capture-starvation` fails
if the scan ever becomes fixed again.

### Health is a statement about now

**An appliance stayed `FAILED` long after the condition had passed**, because
capture loss was judged on the ratio over the appliance's whole life and the
threshold is 1%. A few hundred frames lost while the rule set loaded put the
lifetime figure over the line and kept it there on a machine that had not
dropped a frame since.

Health now judges a rolling one-minute window, below which it reports the
figure as unmeasurable rather than dividing by a handful of frames. The
lifetime ratio is unchanged and still reported — it is the right number to
publish and the wrong one to act on. `make capture-loss-recent` loses traffic
deliberately, confirms health says so, then stops and confirms health notices
that too while the lifetime figure stays high.

### Forwarding: a change made, then measured, then withdrawn

Recorded because the withdrawal is the useful part.

Under the router model the engine is what enables forwarding, so the state it
finds at start-up is whatever the previous shutdown left. That looked like a
poisoning defect, and `cyphorn_forwarding_restore()` was changed to hand
routing back as forwarding ON regardless of the recorded value.

`make forwarding-ownership` refused it, in as many words: *"fail_mode=open left
forwarding enabled for an operator who had it off"*. An operator who
deliberately runs a link non-forwarding gets it back that way, and the
deployment model does not change that.

The poisoning it was reaching for was already solved, and in the right place:
`cyphorn_fail_mode_init()` adopts the originals recorded in the state file
whenever one is present, because those were read before any engine touched the
controls. The live sysctl is trusted only when there is no prior record to
prefer. **The state file is the evidence, not the problem** — deleting it by
hand is what breaks the chain, and it should be removed only when its contents
are known to be garbage, which is the case after the sandbox fault above.

Nothing in the engine changed. The gate was right.

### Three loops that stopped at two ports

The fail-mode state file has recorded a full interface list since multi-port
shipped, and the parsed structure is sized for all of them. Three consumers
still walked a fixed pair — `cyphorn_forwarding_restore()` and both loops in
`fail_mode.c`. Every link past the second was left forwarding by a fail-closed
policy whose purpose is to close it, and skipped by the recovery meant to
reopen it.

### `cyphornctl diagnose`

New. Reads the configuration and the system directly, so it works when the
engine is down — which is when it is needed — and enriches from the control
socket when that answers. Every finding carries the command that fixes it,
including the sandbox directives that hide `/proc/sys` from the engine and the
forwarding state that follows from them.

It also says what the next stop will do, because on a routing appliance
`fail_mode` decides whether the network behind it keeps working and that should
not be a surprise.

### The transit guard

New, and deliberately outside the engine: `/usr/lib/cyphornips/cyphorn-transit-
guard` with its own two units and its own sandbox profile. Nothing inside a
process can be relied on to repair a fault that blinds that process.

It enforces one rule — *while the engine is running, the links it inspects must
carry traffic* — and does nothing at all while the engine is stopped, because
that window belongs to `fail_mode` and closed means closed. It is inert in the
relay model, where forwarding off is correct. `transit_guard=off` disables it.

### Reported

- `cyphornctl status` gained an `Inspected Links:` line carrying each link's
  own capture loss. It replaced `Interfaces: WAN=…, LAN=…`, which named the
  first interface of each list and silently omitted the rest — an appliance
  inspecting four links reported two.
- `capture_loss_recent_ppm` and `capture_loss_recent_measurable` in
  `status json`.
- `transit_guard` is parsed by the engine and acted on by none of it, purely so
  that `cyphornctl config` lists it. That listing is how an operator checks a
  key was understood, and a setting governing whether the network carries
  traffic must not be the one key missing from it.

### The rule parser leaked, and only on the intelligence path

`rule_store_add()` deep-copies every field it keeps, so a caller's template
still owns its own allocations and has to release them. The file loader does,
and has since it was measured leaking four allocations per rule.
`cyphorn_intel_rules_install()` was written the same way and was not corrected
with it, so every intelligence rule leaked its `msg` and its `classtype` — at
start-up and again on every reload, for the life of the process.

Measured under ASan: 320 bytes in 10 allocations, five rules, two strings
each. It is a bounded start-up leak rather than anything an attacker can
drive — the leak is in rule parsing, which reads the operator's rules and the
signed feeds, not the network. `make crash-safety` now passes on all four
malformed-packet corpora with no sanitizer finding.

### Two gates that could not pass a correct engine

Both had been failing long enough to be treated as background noise. Neither
was reporting a defect in the engine.

**`make coverage`** required `coverage_terminal_corrected` to be non-zero in an
offline replay. That counter increments when the eBPF policy map already holds
a DROP the decoder did not decide — a lookup that needs a kernel and a map,
neither of which a replay has. Measured on `p0007_segmentation` with one drop
rule: five packets dropped, every one attributed `INSPECTED_DROPPED` by the
enforcement stage inside the decoder, nothing left to correct. The only way to
satisfy the check was for the engine to mis-attribute a packet first.

The behaviour it was reaching for had no unit coverage at all, so it now has
its own: `tests/test_pipeline_correction.c` asserts the move, both refusals,
that an uninspected packet is never turned into a drop, and that the terminal
counts still sum to one packet afterwards. The replay gate asks only that the
counter be reported.

**`make regression`** compared a digest from the current build against a
baseline recorded by an older one — including `memcap_flows_peak`, a memory
figure. The digest's own rule, written where it is emitted, is that nothing
measured in wall-clock time, run duration or memory use belongs in a compared
value. The flow-domain peak is a reservation sized from `sizeof(Connection)`
times `max_connections`, so a field added to a struct moves it: `ja3-md5`
failed on 32,768 bytes, eight per connection across 4,096 of them, with every
verdict line identical.

Memory figures are now written to the digest without being mixed into its
checksum — the resource-envelopes gate still reads the peak to check that memory held stayed under
the cap, and the determinism gate still compares the whole digest between two runs of one
build. Confirmed before re-recording the baseline: the case still fails against
a build with a deliberately wrong JA3 hash.

### Six unit suites that had stopped being about their subject

All six failed for reasons that had nothing to do with the code they test.

Five — `test_hot_reload`, `test_hot_reload_enforcement`, `test_ui_hot_reload`,
`test_update_system`, `test_update_rollback` — assert bare rule counts after a
reload. `cyphorn_reload_rules()` installs the intelligence rules as well, and a
feature with no list file installs unconditionally, so five rules arrive that
the test never wrote. They were written before that joined the reload path.
They now pin the intelligence features off, which is what
`tests/regression/pinned.conf` already does and for the same reason: a test
about the reload lifecycle should not also be a test of which features ship on
by default.

The sixth, `test_tls_cert_expired_e2e`, had **expired**. Its three fixtures are
defined by offsets from now — valid at −30 days, expired at −30 days,
not-yet-valid at **+30 days** — and were generated once and left on disk.
Thirty days passed. `notBefore` read 22 September 2026 while the suite ran on
the 23rd, so the "not-yet-valid" certificate was valid, rule 4011 matched it
correctly, and the assertion that neither rule fires aborted the run on a
correct engine.

The suite now regenerates the fixtures before reading them, and the assertions
that pinned one generated certificate — its serial, and two literal UTC dates —
are derived from the metadata the engine parsed instead. A formatter that
printed the wrong field or the wrong timezone still fails them; a calendar no
longer does.

### Four gates that were testing the harness, not the engine

- **`negative-parsers`** named `port` as a parser with no negative
  tests, and it was right. `cyphorn_port_role_parse()` reads a word an
  operator wrote, not a packet, but its refusal is load-bearing: a role that
  quietly defaulted would put a link on the wrong side of the trust boundary,
  which makes source-address validation answer backwards and every
  direction-scoped rule fire the wrong way, with nothing saying so. Tested
  rather than excluded — both vocabularies in both cases, thirty-four
  near-misses, and the two NULL arguments. 15 parsers covered, not 14.

- **`telemetry`** named its test interfaces `cyphorn-test-wan`, which
  is sixteen characters. Linux allows fifteen, and the check for that was
  added after the test was written, so the engine refused for the name and the
  test read the absence of the word `telemetry_file` as the key not being
  parsed.

- **`monitor-mode` (E-3)** grepped for `enforcing=no conflict=no`. `required=`
  was inserted between those two fields when the deployment models arrived, so
  the grep had been unsatisfiable ever since while the engine was writing both
  facts it wanted.

- **`control-socket`** was reporting a real defect, and the only one
  of the four: `tests/test_worker_equivalence.sh` started an engine without
  giving it a `telemetry_file` of its own, so it wrote into the appliance's.
  That is the fault that filled a 47 MB `/var/log` on 2026-09-16 and left the
  alert log unwritable for two hours. Fixed in the harness.

### Every capture in the tree now replays identically

A side effect of the maintenance-clock fix, measured rather than assumed:
**all 51 captures in `tests/pcap` produce byte-identical digests across two
runs.** `test_worker_equivalence.sh` carried a comment saying forty of them did
not, and had to choose its captures around that. The comment is corrected and
the selection kept.

### Known, and not fixed here

- A single drain thread is the capacity ceiling. Under a synthetic flood one
  link still loses ~20%, which is that ceiling rather than a defect, and is
  what the threading work addresses.

- **`make performance` fails, and is the one open engine finding.**
  p99 latency is 3,188 us at 64 bytes rising to 4,808 us at 1500, against a
  declared 1,000 us ceiling. That is the same single-threaded inspection path
  seen from the other side, and it is not something a small change closes.

---

## 1.7.0 — 22 September 2026  *(never published; folded into 1.8.0)*

> ### Upgrading from 1.5.0 — read this one
>
> **Country policy has never matched anything on an installed appliance, and
> after this upgrade it will.**
>
> Country policy reads the GeoIP country database, and the database was never
> loaded (see *Fixed*, first entry). Every `drop inbound RU …` and
> `alert outbound CN …` rule you wrote has been inert since you wrote it. The
> engine did not say so beyond one WARN line at start-up that did not name a
> path.
>
> After upgrading, those rules take effect. If you configured country policy
> optimistically — broad rules written on the assumption you could tighten
> them later — **review it before restarting into 1.7.0**, because traffic you
> have been carrying will start being dropped.
>
> The same applies to `SRC_COUNTRY`, `DST_COUNTRY`, `SRC_ASN`, `DST_ASN` and
> `DST_ORG` in `alerts.log`: absent before, present now. Anything parsing that
> file positionally should be checked.
>
> To see what is loaded:
>
>     grep GEOIP /var/log/cyphornips/cyphornips.log | tail -1
>
> `Initialized GeoIP Country (…) and ASN (…)` is the line you want.

### More than two interfaces

**`wan_interface` and `lan_interface` take a list.** Separate the names with
commas or spaces:

    wan_interface = wan, lan1
    lan_interface = wlan0 wlan1 wlan2

Every link in the list is inspected **and enforced on**. Until this release
enforcement attached to the first two ports only; anything past them was
inspected, alerted on, and could not be blocked. That was never a limit in
the mechanism — the TC program does not read `skb->ifindex` and the policy map
is keyed on the flow tuple alone — it was a struct that spelled out two ports.

`port = <interface>:<inside|outside>` still works and is the form to use when
a link's role is not the role of the side it would be listed under.

Sixteen ports at most, both sides counted. An interface may appear once. A
name longer than Linux allows is refused by name rather than truncated into
one nobody wrote.

Forwarding and `fail_mode` follow the same list. Enforcing on five links while
forwarding was owned on two would be an appliance failing closed on two of
them and open on the rest, while reporting itself fail-closed.

### Nothing is conditioned on the hardware

Two places decided behaviour from what they guessed the hardware was, and both
were silent about it:

- Country-policy direction was decided by searching the interface name for
  `wan` or `lan`. On a server — `eno1`, `enp3s0`, `ens18` — none of those
  substrings appears, so the step answered nothing and fell through to address
  heuristics without saying so. It now reads the configured role of the
  arrival port.
- The source-address-validation ingress role was derived from a port's
  *position*: index 0 is the WAN, everything else the LAN. A second uplink
  written `port = eth3:outside` would have been labelled inside, and SAV —
  whose whole question is whether a packet arrived on the side its address
  claims — would have answered backwards on every packet crossing it.

`make server-portability` holds this: the engine runs on links named the way a
distribution names them and reaches the same conclusions.

### Coverage: the equation has a term for packets in flight

`appliance_invariant` read VIOLATED under load and recovered afterwards.
Measured on a four-port unit: bursts of 13,000 to 33,000 packets between
samples put it out by +1,289 and +2,307, returning to a constant −2 each time.

The interface counts a packet when it lands in the capture ring; the engine
counts it when it takes it out. Between those instants the packet is real,
lost by nobody, and in neither number. `in_flight` is that population,
**measured** by walking the ring's block descriptors — not derived from the
difference the equation checks, which would make the check vacuous.

Two new settings, both `0` = derive:

| Key | Default | What it does |
|---|---|---|
| `coverage_tolerance_per_port` | derive (64/port) | How far the equation may be out per inspected link |
| `coverage_violation_samples` | 3 | Consecutive breaches before VIOLATED is declared |

The second is what a tolerance cannot do. A burst and a shortfall look
identical in one sample and differ in the next: the burst heals, the shortfall
grows. Requiring a breach to repeat catches a persistent shortfall of seventy
packets, which no tolerance wide enough to cover a burst ever would. Every
delta is printed either way and an unconfirmed breach is logged at WARN — only
the verdict waits.

Measured after: the same bursts leave +5 to +66.

### Fixed

- **GeoIP and ASN enrichment now work on an installed appliance.** They never
  had. The compiled default paths pointed at
  `/root/cyphorn-engine/geoip/dbip-country-lite-2026-08.mmdb` — this
  repository's working copy, with the dated filenames it carries — while the
  package installs to `/etc/cyphornips/geoip/dbip-country-lite.mmdb`. A
  different directory and a different filename, so neither file was ever
  found. The engine logged `No MMDB databases found or failed to load` at WARN
  and carried on. **See the upgrade note below: this changes behaviour you may
  be relying on.** The warning now names the paths it tried and says what
  their absence costs.
- **A `pass` rule no longer reopens a flow already ruled DROP.** The sticky
  flow verdict was conditioned on nothing having matched the packet, so a
  packet that *did* match released itself. A flow dropped on its SYN carried
  on being dropped until one packet matched a `pass` and was let through — one
  packet out of an enforced flow, logged as `ALLOW pass` with no sign the flow
  around it was closed. `pass` on a flow that was never dropped is unchanged.
- The relay path compared the arrival against two port constants and dropped
  anything else with "invalid packet ingress" — in the one deployment model
  where the engine itself carries transit.
- `cyphorn_enforcement_is_local_address()` kept only the inspected pair as its
  fallback. Two links out of five would have answered "not local" for the
  appliance's own address on the other three, and a port-wildcard drop keyed
  on one of those takes out every NAT'd client behind it.
- `cyphornctl config` now lists every inspected link. It printed the primary
  alone, under a header promising the values in force.
- The control socket accepted `config` and `config json`, and the error text
  listing supported commands named neither, nor `explain`.
- `telemetry_max_size` and `telemetry_max_files` were documented in the
  shipped config and nowhere else.

### Packaging

- The package version is read from `CYPHORN_ENGINE_VERSION` and the
  architecture from `dpkg --print-architecture`. Both were constants, two
  releases stale, and a build on a non-amd64 host would have produced a
  correctly-built binary in a package declaring the wrong architecture. Cross
  builds are explicit: `ARCH=amd64 CC=x86_64-linux-gnu-gcc ./packaging/build_deb.sh`.
- The package ships `/usr/share/doc/cyphornips/copyright` and `SECURITY.md`.
  It previously shipped no copyright file at all.

### Licence and policy

- `LICENSE` carries the CyphornIPS License. `LICENSE` was zero bytes and
  `README.md` advertised **Apache 2.0 / MIT** — on proprietary software that
  forbids redistribution, modification and reverse engineering.
- `SECURITY.md` gains a supported-versions table, GitHub private vulnerability
  reporting as the preferred channel, a CVE policy, and a denial-of-service
  scope that says DoS is **in** scope: an inline IPS that can be switched off
  by the traffic it inspects has failed at its purpose. Overwhelming the
  hardware with volume remains out of scope — that is arithmetic.
- The documentation gains section 41, *Licence & permitted use*, including the
  thing most readers get wrong: running this on your own employer's network is
  commercial use whether or not you charge anyone.

### Testing

Four new gates: `interface-list`, `multiport-enforcement`,
`server-portability`, `coverage-persistence`. `tests/test_engine_config.c`
existed but no Makefile target built it, so it had not run in some time.

Three gates were failing against their own models rather than against the
engine, and the models were corrected:

- `race-detection` grepped for the string `pthread_create` and counted a
  mention in a comment as a thread.
- `fast-pattern` did not model the case — a rule whose backward span cannot be
  bounded is not indexed and is a candidate for every packet. 19,343
  "extra" selections were the engine applying it.
- `management-plane` was right: `config` really was undocumented.

**Known:** running the suite on a live appliance gives about three times more
failures than are real — the gates start their own engine on the service's own
runtime paths. Stop the service before judging a result.

---

## 1.5.0 and earlier

No changelog was kept. `git log` is the record.
