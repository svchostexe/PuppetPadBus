# Scoping a DualSense (DS5) emulation target

Status: **scoping only — no implementation.** Written 2026-08-13 to decide whether
adding a Sony DualSense target to this fork is worth doing, and what it would cost.

## Short version

The driver-side work is well-bounded and follows a path the DualShock 4 target
already paved. The prerequisite used to be the blocker: **every consumer reaches
this bus through ViGEmClient**, and a DS5 target is useless until that library can
carry new API. That is now resolved — `sdk` points at `svchostexe/ViGEmClient`, a
fork we control, so client-side changes can actually land.

The second-biggest cost is not code either — it is the DualSense's **output**
report, which carries adaptive-trigger effect parameters. That is genuinely new
surface with no analogue in the DS4 target.

## Why the driver side is tractable

The DualSense presents to Windows as a plain HID device. There is no class driver
equivalent to `XUSB.sys` in the path, so emulation follows the DS4 model
(`sys/Ds4Pdo.cpp`) rather than the considerably fiddlier XUSB one — no boot
sequence of magic interrupt blobs, no `_InterruptInitStage` state machine, no
vendor control transfer at `SetupPacket[6] == 0x04`.

`EmulationTargetPDO` is already the right abstraction. A DS5 target is a new
subclass; nothing in the base class needs to change.

## Work items

### 1. Extend ViGEmClient — done as a prerequisite, work still to do

`.gitmodules` points `sdk` at `https://github.com/svchostexe/ViGEmClient.git`, a
fork of the archived upstream, pinned at `cb8c9f4` (`v1.17.188.0-27-gcb8c9f4`).
New API can land there now. Nothing has been added yet.

New public API, mirroring the DS4 entry points:

- `vigem_target_ds5_alloc()`
- `vigem_target_ds5_update()` / `..._update_ex()`
- `vigem_target_ds5_register_notification()` / `..._unregister_notification()`
- `vigem_target_ds5_await_output_report()` / `..._timeout()`

### 2. Shared definitions — `sdk/include/ViGEm/Common.h` and `km/BusShared.h`

- `VIGEM_TARGET_TYPE` gains `DualSenseWired = 3`. Values `0` and `2` are taken;
  `1` was deliberately skipped for compatibility, so do not reuse it.
- New structs: `DS5_REPORT`, `DS5_SUBMIT_REPORT`, `DS5_OUTPUT_REPORT`,
  `DS5_OUTPUT_BUFFER`, `DS5_AWAIT_OUTPUT`, plus `_INIT` helpers, following the DS4
  shapes at `Common.h:95-263` and `BusShared.h:333-506`.
- New IOCTLs. `IOCTL_VIGEM_BASE + 0x204/0x205` are reserved-and-commented for XGIP
  (`BusShared.h:71-72`); take `0x208` onward rather than reusing them:
  `IOCTL_DS5_SUBMIT_REPORT`, `IOCTL_DS5_REQUEST_NOTIFICATION`,
  `IOCTL_DS5_AWAIT_OUTPUT_AVAILABLE`.
- **Do not change `GUID_DEVINTERFACE_BUSENUM_VIGEM`.** The header comment tells
  forks to change it, but this fork stays wire-compatible with existing
  ViGEmClient consumers on purpose. `VIGEM_COMMON_VERSION` (`BusShared.h:49`) is
  the correct lever if a compatibility break is ever needed — note that bumping it
  makes older clients skip the bus entirely.

### 3. New PDO class — `sys/Ds5Pdo.hpp` / `sys/Ds5Pdo.cpp`

Subclass `Core::EmulationTargetPDO` and implement the pure virtuals declared in
`sys/EmulationTargetPDO.hpp:82-197`:

| Method | Notes |
| --- | --- |
| `PdoPrepareDevice` | Hardware/compatible IDs. Default VID/PID `054C:0CE6`. |
| `PdoPrepareHardware` | Seed default input report, start the flush timer. |
| `PdoInitContext` | Timer + per-serial registry state, as DS4 does. |
| `GetConfigurationDescriptorType` | Config descriptor blob. |
| `UsbGetDeviceDescriptorType` | Device descriptor. |
| `SelectConfiguration` / `UsbSelectInterface` | Pipe and interface fixups. |
| `UsbGetDescriptorFromInterface` | HID report descriptor — the big blob. |
| `UsbGetStringDescriptorType` | "Sony Interactive Entertainment" / "Wireless Controller". |
| `UsbClassInterface` | HID GET/SET_REPORT feature reports. |
| `UsbBulkOrInterruptTransfer` | Input reports out, output reports in. |
| `UsbControlTransfer` | DS4's version only needs to stall two USB 1.0 cases. |
| `SubmitReportImpl` | Copy submitted report into the pending URB. |
| `ProcessPendingNotification` | Drain the output buffer queue to waiters. |
| `AbortPipe` | Stop the timer. |

### 4. Wiring

- `sys/busenum.cpp:140-185` — add `case DualSenseWired:` to **both** switch blocks
  (the default-IDs one and the explicit VID/PID one). Missing the second is an easy
  bug to introduce.
- `sys/Queue.cpp` — three handlers modelled on `Bus_Ds4SubmitReportHandler`
  (including its input-size validation, which is the good example to copy) and
  `Bus_Ds4RequestNotificationHandler`.
- `sys/Driver.cpp:62-74` — three rows in `ViGEmBus_IoctlSpecification[]`. DMF
  enforces the declared minimum buffer sizes, so get them right here.
- `sys/PuppetPadBus.vcxproj` + `.filters` — add the new sources.
- `Bus_PlugInDevice` also calls `SetOutputReportNotifyModule` for DS4 only
  (`busenum.cpp:198-201`); DS5 needs the equivalent if it supports await-output.

## The hard parts

**Adaptive triggers.** The DualSense output report carries per-trigger effect
descriptors (mode plus parameter bytes for resistance, weapon, vibration effects).
This is the real new work. Two viable designs:

1. *Passthrough* — surface the raw output report to userland via the
   await-output mechanism and let the client interpret it. Cheapest, most
   flexible, matches how `DS4_AWAIT_OUTPUT` already works.
2. *Structured* — model trigger effects in `DS5_OUTPUT_REPORT`. Friendlier API,
   but bakes Sony's report layout into the ABI, which is a poor trade for a
   driver that has to keep a stable interface.

Recommend passthrough, with a structured convenience layer in the client library
if wanted.

**The HID report descriptor.** The DS4's is 467 bytes hand-transcribed into
`Ds4Pdo.cpp:647-879`. The DualSense's is larger. It must be transcribed exactly —
a wrong byte here produces a device that enumerates but reports garbage axes, and
the failure is tedious to debug.

**Audio interfaces.** A real DualSense exposes USB audio class interfaces
alongside HID for the headset jack and speaker. Emulating them is a large amount
of extra work for little benefit; most consumers key off VID/PID and the HID
interface alone. Recommend omitting them and documenting that the emulated device
is HID-only — but expect that anything probing for the audio endpoints will notice.

**Testing.** There is no automated test path for any of this. Validation means
installing the driver on a test machine and checking enumeration, then input
through something that consumes DualSense HID directly, then output/rumble, then
trigger effects. Budget for that, and note the driver must be test-signed to load.

## Recommendation

Sequence it as: shared struct and IOCTL definitions → PDO class with input reports
only → output/rumble → adaptive triggers via passthrough. Each stage is
independently testable, and the first is a prerequisite for anything else.

The ViGEmClient fork that used to gate all of this now exists, so the remaining
question is appetite rather than feasibility.

## Related

For comparison, an Xbox One / XGIP target would be **harder**, not easier, despite
the reserved IOCTL slots: it needs GIP protocol emulation rather than plain HID,
and the Windows-side consumer story is murkier.
