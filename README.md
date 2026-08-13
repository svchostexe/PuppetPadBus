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

**There are no pre-built signed releases.** You build it yourself, and running the result requires taking your machine out of its default security posture.

Concretely: you must disable Secure Boot, enable test signing, and sign the driver with your own certificate — Windows will not load an unsigned kernel driver, and test signing widens *which* signatures are accepted rather than removing the requirement. Once enabled, **any** test-signed driver can load on that machine, not just this one.

**[docs/installing.md](docs/installing.md) covers the whole procedure**, including uninstall and the failure modes worth recognising. Read it before you start. If that trade-off is not acceptable to you — and it reasonably might not be on a machine you care about — this driver is not currently something you should install.

Production signing requires an EV code-signing certificate and Microsoft attestation signing through Partner Center. If that changes for this project, it will be announced in the releases.

## Security

This fork ships **no auto-updater**. The original bundled an Advanced Installer update component that polled `updates.vigem.org` on a daily scheduled task and could download and launch an installer. That domain was transferred away from the original maintainer in December 2023, so prompts from it can no longer be attributed to the original developer. The update component, its scheduled task and its endpoint have all been removed from the installer project here. Nothing in a PuppetPad build contacts the network on its own.

**If you have an existing ViGEmBus installation**, that updater may still be present and active on your machine. Disabling or removing it is worthwhile regardless of whether you switch to this fork.

To report a security problem in this driver, see [SECURITY.md](SECURITY.md). Please do not open a public issue.

## License

BSD-3-Clause, inherited from the original project. See [LICENSE](./LICENSE).

Copyright on the original work remains with Nefarius Software Solutions e.U. and contributors; those notices are retained throughout the source, as the license requires. See [NOTICE](./NOTICE) for the fork's provenance.

## How to build

### Getting the source

The client library is a submodule, so clone recursively:

```
git clone --recurse-submodules https://github.com/svchostexe/PuppetPadBus.git
```

If you already cloned without it: `git submodule update --init`.

### Prerequisites

- [Visual Studio 2019](https://docs.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads#step-1-install-visual-studio), including the build tools for every architecture you intend to target — ARM64 needs `VC.Tools.ARM64` and `VC.Runtimes.ARM64.Spectre`, which are not installed by default
- [WDK for Windows 10, version 2004](https://docs.microsoft.com/en-us/windows-hardware/drivers/other-wdk-downloads#step-2-install-the-wdk) (`10.0.19041.0`)
- [Driver Module Framework (DMF)](https://github.com/microsoft/DMF), cloned separately:

```
git clone https://github.com/microsoft/DMF.git
cd DMF
git apply --reject <path to this repo>\patches\dmf.diff
msbuild Dmf.sln /p:Configuration=Release /p:Platform=x64
```

DMF must be built once per architecture before the driver will link against it. Note that DMF calls the 32-bit platform `Win32` where this solution calls it `x86`, and that only `Release` is supported — DMF produces no Debug output, so a Debug build of the driver fails at link.

### Building

Build the **solution**, not the project — `sys/PuppetPadBus.vcxproj` resolves includes through `$(SolutionDir)` and fails to find them when built standalone.

```
set DmfRootPath=<path to your DMF clone>
msbuild PuppetPadBus.sln /p:Configuration=Release /p:Platform=x64
```

`Platform` accepts `x64`, `x86` or `ARM64`. `DmfRootPath` must point at the **root of the DMF clone**, not its parent — the default in `sys/Dmf.props` assumes a different layout and will fail with a misleading "cannot open include file `DmfModules.Library.h`".

Output lands in `bin\<arch>\`, alongside the generated catalog.

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
