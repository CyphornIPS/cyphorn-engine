# Is there a way to verify the license in CyphornIPS?

**No. There is no license key, no activation, no expiry, and no check of any
kind in the engine.** This is a decision, not an omission, and it is worth
writing down before somebody adds one.

---

## What the license actually is

The license is a **legal** instrument, not a technical one. LICENSE §2 says
commercial use requires a signed agreement. That obligation exists whether or
not the software can tell. A company using CyphornIPS to protect its network
without an agreement is in breach of a license it accepted by installing —
and that is enforceable through the ordinary route, which is a letter.

---

## Why not build a check in

**1. It cannot work on this product.**

CyphornIPS ships as source you can build. Anyone who wants to remove a check
deletes the `if` and rebuilds. Even a compiled-only distribution is one
`objdump` away. Every serious commercial user — the one worth a license fee —
has the skill to strip it, and will if it gets in the way. The check would
only ever be met by the honest, who did not need it.

**2. It attacks the thing the product is for.**

This is a security appliance in the path of a network. A license check is a
new failure mode on that path: a clock that drifts, a file that goes missing,
a call to a server that is down, and traffic stops or goes uninspected. An IPS
that stops protecting because of a licensing question has failed at the only
job it has.

Anything with an expiry date will eventually take a network down at 3am. That
is a certainty, not a risk.

**3. It contradicts the product's own argument.**

CyphornIPS's claim is that it can prove what it inspected and say what it
missed. It publishes a coverage equation that fails loudly when it does not
balance. A hidden check that changes behaviour based on something other than
the traffic is the opposite of that — it is exactly the class of silent,
unexplained behaviour the engine's design exists to make impossible.

**4. Phoning home is worse than the problem.**

A license server means the appliance calls out. On a security device that is
an outbound connection you now have to justify to the same security team you
are selling to, on a schedule you control and they do not. It is a reason to
refuse the product.

---

## What to do instead

**Make the obligation impossible to miss, and asking easy.**

- LICENSE §2 defines commercial use broadly and explicitly — including
  protecting your own employer's infrastructure, which is the case most people
  assume is fine.
- The README says it in plain words with the contact address, above the fold.
- Section 2 ends with: *"Asking costs nothing and is answered."*

**Then rely on the two things that actually work:**

- **Visibility.** Companies using unlicensed software are found by their own
  staff, their auditors, or a support request that reveals the deployment.
- **A reason to be licensed.** Support, a written agreement to point an auditor
  at, and someone to call. Those are what a business pays for. A company large
  enough to matter is not deterred by a key check; it is moved by wanting a
  contract on file.

---

## If you decide to add one anyway

Then be honest about which of these you are building:

| | What it does | Cost |
|---|---|---|
| **A banner** | Logs an unlicensed-use notice at start-up and on the status surface. Changes no behaviour. | None. Honest users see it; nobody's traffic is at risk. |
| **A feature gate** | Some capability is off without a key. | Users run the free build. You now maintain two products. |
| **A kill switch** | Stops inspecting, or stops forwarding, without a valid license. | **Do not.** This is the 3am outage, and on a `fail_mode=closed` appliance it takes the network with it. |

The banner is the only one of the three that does not compromise the product,
and it is a real option: a line in `cyphornctl status` reading *"Licensed for
personal and evaluation use. Commercial use requires an agreement —
info@cyphorn.com"* costs nothing, cannot break anything, and reaches
every operator who runs the command.

If you want that, it is a small, contained change to the status surface, and
nothing on the packet path needs to know about it.
