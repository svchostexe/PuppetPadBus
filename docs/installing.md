# Installing a self-built PuppetPadBus

This project publishes **source, not signed binaries**. Installing what you build
means taking your machine out of its default security posture. Read this section
before the commands.

## What you are agreeing to

Windows will not load a kernel-mode driver that lacks an acceptable signature.
There are two separate hurdles, and people routinely conflate them:

1. **Signature enforcement.** A completely unsigned driver never loads. Test
   signing mode does not remove this requirement — it widens *which* signatures
   are acceptable, so a driver signed with your own self-made certificate is
   allowed. You still have to sign it.
2. **Secure Boot.** Test signing cannot take effect while Secure Boot is
   enabled. `bcdedit` may appear to accept the change and the setting is then
   ignored, which is a confusing way to lose an afternoon. You must disable
   Secure Boot in firmware.

Disabling Secure Boot and enabling test signing means **any** test-signed driver
can load on that machine, not only this one. That is a real and lasting
reduction in your defences. Do it on a machine where you accept that, ideally a
VM or a dedicated test box — not casually on a daily driver.

If that trade is not acceptable to you, stop here. There is currently no signed
release, and there is no safe way to load an unsigned kernel driver.

## 1. Enable test signing

Disable Secure Boot in your firmware first. Then, in an **administrator** prompt:

```
bcdedit /set testsigning on
```

Reboot. You will see a "Test Mode" watermark on the desktop afterwards; that is
expected and is how you know it took.

To undo everything later:

```
bcdedit /set testsigning off
```

then re-enable Secure Boot.

## 2. Build with test signing enabled

The project ships with `SignMode` set to `Off`, so a plain build produces an
**unsigned** driver that will not load. Build with test signing instead, which
makes the WDK sign both the driver and its catalog, in that order:

```
msbuild PuppetPadBus.sln /p:Configuration=Release /p:Platform=x64 /p:SignMode=TestSign
```

This generates and uses a test certificate. If you sign by hand instead, note
the ordering trap: the catalog stores hashes of the files it covers, so the
`.sys` must be signed **before** the catalog is generated, and the catalog
signed after. Signing the `.sys` after `Inf2Cat` has run leaves a catalog that
no longer matches and an install that fails with an unhelpful error.

The test certificate must also be trusted by the machine you install on. Export
it and import it into both **Trusted Root Certification Authorities** and
**Trusted Publishers** in the *local machine* store — not the user store.

## 3. Install the driver

The bus is a root-enumerated device, so adding the driver package is not enough
on its own — something has to create the device node. `pnputil` cannot; `devcon`
can, and ships with the WDK:

```
C:\Program Files (x86)\Windows Kits\10\Tools\x64\devcon.exe
```

From an **administrator** prompt, in the output directory (`bin\x64\PuppetPadBus\`):

```
devcon.exe install PuppetPadBus.inf "PuppetPad\PuppetPadBus\Gen1"
```

That both stages the driver package and creates the root device node.

## 4. Verify

The device appears in Device Manager under **System devices** as *PuppetPad
Virtual Gamepad Emulation Bus*. From the command line:

```
pnputil /enum-devices /class System /connected
sc query PuppetPadBus
```

A healthy install shows the service running and the device with no problem code.

## 5. Uninstall

```
devcon.exe remove "PuppetPad\PuppetPadBus\Gen1"
```

Then remove the staged driver package. Find its assigned `oemNN.inf` name first,
since Windows renames it on staging:

```
pnputil /enum-drivers
pnputil /delete-driver oemNN.inf /uninstall
```

## Troubleshooting

**The driver does not start, code 52.** "Windows cannot verify the digital
signature." Either the driver is unsigned (you built without
`/p:SignMode=TestSign`), the test certificate is not in the local machine's
trusted stores, or test signing is not actually active because Secure Boot is
still on.

**The bus installs but immediately stops.** Check whether the original ViGEmBus
is still present. Both claim the same device interface GUID, and the driver
deliberately refuses to start when another bus already exposes it. Uninstall
ViGEmBus first — the two cannot coexist.

**`devcon` is missing.** It is a WDK component, not a Windows one. Use the copy
matching your architecture from `Windows Kits\10\Tools\`.

**Nothing detects the virtual controller.** The bus driver only provides the
bus. Whatever creates gamepads on it — DS4Windows, Sunshine and similar — talks
through the ViGEmClient library, so check that side too.

## Logs worth collecting

If you report an installation problem, attach `C:\Windows\INF\setupapi.dev.log`.
It records every step of the device install and is usually decisive.
