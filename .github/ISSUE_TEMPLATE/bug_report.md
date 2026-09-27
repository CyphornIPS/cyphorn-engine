---
name: Bug report
about: Something the engine did that it should not have, or did not do that it should
title: ''
labels: bug
---

## What happened

<!-- What you saw. Exact output beats a description of it. -->

## What you expected

## How to reproduce

<!-- If it needs specific traffic, say what kind. -->

## Your appliance

Please paste the output of these three, which answer most of the questions
a maintainer would otherwise have to ask:

```
cyphornctl diagnose
cyphornctl config | head -40
cyphornctl status
```

<!--
diagnose works even if the engine is down, which is when it is most useful.
If the engine will not start at all, its output plus the last 30 lines of
/var/log/cyphornips/cyphornips.log is usually enough to find the cause.
-->

## Version

<!-- cyphornctl --version, and how you installed (deb / built from source) -->
