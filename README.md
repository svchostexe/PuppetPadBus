# PuppetPad Bus Driver

Windows kernel-mode driver emulating well-known USB game controllers.

**A community-maintained fork of [ViGEmBus](https://github.com/nefarius/ViGEmBus), which was retired in November 2023.**

---

## What this is

`PuppetPadBus` is an independent continuation of the archived `nefarius/ViGEmBus` kernel driver. It exists because a large number of tools — input remappers, remote play and game streaming clients, accessibility software — still depend on that driver, and it stopped receiving fixes when upstream shut down.

This fork is **not affiliated with, endorsed by, or supported by** Nefarius Software Solutions e.U., ViGEM GmbH, or any contributor to the original project. Please do not take support requests for this fork to them.

The name was changed deliberately. Upstream retired after a registered trademark conflict, and this project does not use the original mark in its name, branding, package identity or driver strings. References to "ViGEmBus" here are factual, describing what this software was forked from.

### What it keeps

The device interface GUID and the IOCTL interface are **unchanged**, so existing software built against ViGEmClient works against this driver without modification. That compatibility is the entire point of the fork.

### What it does not do

- **No signed binaries.** This repository publishes source only. See [Installation](#installation).
- **No new emulated devices yet.** Xbox 360 and DualShock 4, same as upstream. A DualSense target has been [scoped](docs/dualsense-target-scoping.md) but not started.
- **No auto-updater.** See [Security](#security).

## ⚠️ It cannot coexist with ViGEmBus

Both drivers expose the same device interface GUID, and the bus refuses to start if another one is already present. **Uninstall ViGEmBus before installing this.** Running both is not a supported configuration and will simply result in one of them failing to load.

## Emulated devices

- [Microsoft Xbox 360 Controller](https://en.wikipedia.org/wiki/Xbox_360_controller)
- [Sony DualShock 4 Controller](https://en.wikipedia.org/wiki/DualShock#DualShock_4)

## Supported systems

Windows 10 and 11, on x86, x64 and ARM64. Windows Server is untested and unsupported. Anything older than Windows 10 is not supported.

## Installation

**There are no pre-built signed releases.** You build it yourself.

This matters more than it sounds. Windows will not load an unsigned kernel-mode driver on a normal x64 install. To run a self-built driver you must enable test signing:

```
bcdedit /set testsigning on
```

and reboot. That disables a meaningful security control on your machine — it permits *any* test-signed driver to load, not just this one. Understand that trade-off before you do it. If you are not comfortable with it, this driver is not currently something you should install.

Production signing requires an EV code-signing certificate and Microsoft attestation signing through Partner Center. If that changes for this project, it will be announced in the releases.

## Security

This fork ships **no auto-updater**. The original bundled an Advanced Installer update component that polled `updates.vigem.org` on a daily scheduled task and could download and launch an installer. That domain was transferred away from the original maintainer in December 2023, so prompts from it can no longer be attributed to the original developer. The update component, its scheduled task and its endpoint have all been removed from the installer project here. Nothing in a PuppetPad build contacts the network on its own.

**If you have an existing ViGEmBus installation**, that updater may still be present and active on your machine. Disabling or removing it is worthwhile regardless of whether you switch to this fork.

To report a security problem in this driver, see [SECURITY.md](SECURITY.md). Please do not open a public issue.

## License

BSD-3-Clause, inherited from the original project. See [LICENSE](./LICENSE).

Copyright on the original work remains with Nefarius Software Solutions e.U. and contributors; those notices are retained throughout the source, as the license requires. See [NOTICE](./NOTICE) for the fork's provenance.

## How to build

### Prerequisites

- [Visual Studio 2019](https://docs.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads#step-1-install-visual-studio)
- [WDK for Windows 10, version 2004](https://docs.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads#step-2-install-the-wdk) (`10.0.19041.0`)
- [Driver Module Framework (DMF)](https://github.com/microsoft/DMF), cloned separately, with `patches/dmf.diff` applied

### Building

Build the **solution**, not the project — `sys/PuppetPadBus.vcxproj` resolves includes through `$(SolutionDir)` and fails to find them when built standalone.

```
set DmfRootPath=<path to your DMF clone>
msbuild PuppetPadBus.sln /p:Configuration=Release /p:Platform=x64
```

`Platform` accepts `x64`, `x86` or `ARM64`. Only the `Release` configuration is supported — DMF is built for Release only, so a Debug build fails at link. DMF must be built once per architecture before the driver will link against it, and note that DMF itself uses `Win32` where this solution uses `x86`.

Output lands in `bin\<arch>\`.

## Contributing

Bug reports and pull requests are welcome on the [issue tracker](../../issues). Security issues go through [SECURITY.md](SECURITY.md) instead.

Please respect that the issue tracker is not a helpdesk, and that this project has no relationship with upstream's support channels.

## Compatible software

Software written against the ViGEmBus client API works with this driver unmodified. A non-exhaustive list of projects known to build on that API — **listed for compatibility reference only; none of these endorse or are affiliated with this fork**:

- [DS4Windows](https://ryochan7.github.io/ds4windows-site/)
- [Sunshine](https://github.com/LizardByte/Sunshine)
- [GloSC](https://github.com/Alia5/GloSC)
- [UCR](https://github.com/Snoothy/UCR)
- [XOutput](https://github.com/csutorasa/XOutput)
- [BetterJoy](https://github.com/Davidobot/BetterJoy)
- [Touchmote](https://github.com/Ryochan7/Touchmote/tree/ryochan7)
- [XJoy](https://github.com/sam0x17/XJoy)
- [RdpGamepad](https://github.com/microsoft/RdpGamepad)
- [NetJoy](https://github.com/Qcent/NetJoy/)
