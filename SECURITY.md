# Security Policy

## Scope

This project is a Windows **kernel-mode** driver. Defects here can bugcheck a machine or, in the worst case, be reachable from unprivileged user mode through the driver's IOCTL interface. Reports are taken seriously and handled privately.

**Know what you are auditing.** This is a personal, as-is fork maintained by one person who is not a Windows kernel developer, and its changes were written with heavy AI assistance. They have not had expert review. That makes independent scrutiny more valuable here than it would be on a project with a security team behind it — but it also means you should assume defects are present rather than assume they have been caught. See the project status section in the [README](README.md).

## Reporting a vulnerability

**Do not open a public issue for security problems.**

Use GitHub's private vulnerability reporting on this repository (Security → Report a vulnerability), which keeps the report visible only to maintainers.

Please include:

- what the defect is, and which source file and function it lives in
- how it is reached — in particular whether it is reachable from user mode, and whether it needs administrative rights
- what happens when it goes wrong (bugcheck, pool corruption, information disclosure, hang)
- a reproducer if you have one, or the reasoning that led you to it if you do not

Reports are read and taken seriously, but this is a spare-time project with no on-call rotation behind it — there is no guaranteed response time, and a reply may take a while. If a report is confirmed, the fix and an advisory will be published together. If a report needs kernel expertise beyond mine, I will say so plainly rather than sit on it.

## Scope notes

**In scope:** anything in `sys/`, the IOCTL interface defined in the shared headers, and the installer project in `setup/`.

**Not in scope:** the requirement to enable test signing in order to load a self-built driver. That is a documented consequence of this project not shipping signed binaries, not a vulnerability. See the README.

**Upstream defects:** this driver is a fork of the retired [ViGEmBus](https://github.com/nefarius/ViGEmBus). Many defects here are inherited rather than introduced by the fork. If you find one that also affects the original, please say so in the report — other forks and existing ViGEmBus installations may be affected too, and coordinating disclosure is the right thing to do. Note that upstream is archived and is not issuing fixes.

## Supported versions

Only the current `master` branch is supported. There is no long-term support branch, and no backporting to older releases.
