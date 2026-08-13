---
name: Setup issues
about: Use this if you have problems installing the driver

---

## Read first

Before reporting an installation problem, check these:

- **Are you on Windows 10 or 11?** Anything older is not supported. That is intentional, please do not report it.
- **Did you build for the right architecture?** x86, x64 and ARM64 builds are not interchangeable.
- **Is the original ViGEmBus still installed?** PuppetPadBus and ViGEmBus cannot run at the same time — both claim the same device interface, and whichever loads second refuses to start. Uninstall ViGEmBus first.
- **Is the driver signed?** This project publishes source, not signed binaries. A driver you built yourself will not load on a normal Windows install unless you enable test signing (`bcdedit /set testsigning on`, then reboot). Understand what that does to your machine's security posture before you turn it on.

## Still stuck

Attach these log files. Issues without them will be closed.

### Setup log

Run the setup from `cmd` or PowerShell with `/L*V .\install.log` appended. That writes `install.log` next to the setup. Attach it after the install fails.

### Windows device install log

Attach `C:\Windows\INF\setupapi.dev.log` as well.

Compress both if upload gives you trouble.
