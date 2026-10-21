# Running Dasharo (coreboot) firmware inside VirtualBox

## Problem

We would rather boot our guests under real Dasharo firmware than VirtualBox's own
edk2-derived EFI. Dasharo is an open, auditable firmware stack; VirtualBox's EFI is
a stock OVMF fork carried as VirtualBox-internal source. The question: can Dasharo be
made to run as the firmware of a VirtualBox guest, and if so, what is the smallest,
cleanest change that achieves it?

This note records the architecture finding, the change we made, the proof it works,
and the clean next step. The two code deliverables live in their own repos:

- `org-ai-assisted/virtualbox` -- the VirtualBox-side change (a `DevEFI` loader path).
- `org-ai-assisted/dasharo` -- the coreboot-side firmware config, proof, and plan.

## What Dasharo actually is (the pivotal fact)

Dasharo is a **coreboot distribution**, not a standalone OVMF build:

- **coreboot** does the silicon/platform init (bootblock, romstage, ramstage).
- **`DasharoPayloadPkg`** is a `UefiPayloadPkg` fork built as a coreboot *payload*;
  it provides the UEFI environment, Secure Boot, and setup UI, and is board-agnostic
  because it reads the coreboot table at runtime.

So "porting Dasharo to VirtualBox" is not "build an OVMF variant". It is "get coreboot
to run on VirtualBox's virtual platform, then load the Dasharo payload on top". That
reframes the whole problem onto two independent pieces: a firmware image (coreboot)
and whatever it takes for the hypervisor to load that image.

## Why VirtualBox cannot load it as-is

VirtualBox's EFI device (`src/VBox/Devices/EFI/DevEFI.cpp`) accepts only an
OVMF-shaped combined flash descriptor:

- `efiParseFirmware` requires a varstore-first firmware volume whose first FV carries
  `gEfiSystemNvDataFvGuid` (`FFF12B8D-7696-4C8B-A985-2747075B4F50`, the standard edk2
  GUID -- not a VirtualBox-proprietary one), followed by a main FV with a reset
  vector. Anything else is rejected at parse time.
- A `coreboot.rom` is not an edk2 firmware volume. It is a flat image: the reset
  vector sits at the very top of the 4 GiB physical address space (`0xFFFFFFF0`), and
  the body is CBFS plus an FMAP. `efiParseFirmware` refuses it.

Therefore a real-coreboot port needs a VirtualBox-side change. There is no way to
shape a coreboot image to satisfy `efiParseFirmware` without ceasing to be coreboot.

## The change: a raw-image loader path in DevEFI (ring-3 only)

We added one CFGM boolean, `CorebootRom` (default off). When set, `efiLoadRom`:

- skips `efiParseFirmware`;
- aligns the image to a page, sets `offEfiRom = 0`, and lets the existing map loop map
  the whole image as shadowed ROM at `[4G - size, 4G)`, so the reset vector lands at
  `0xFFFFFFF0` exactly as coreboot expects;
- instantiates a small (4 KiB) **vestigial flash device** just below the ROM.

The key design constraint was discovered the hard way: the change had to be **ring-3
only, with no change to the shared `DEVEFI` device struct**.

- Adding a field to the shared struct changes `cbInstanceShared`, and PDM then rejects
  the device at ring-0 registration with `VERR_INCOMPATIBLE_CONFIG` ("Ring-3 does not
  match ring-0 device registration"). The ring-0 device module is also cached by name
  across processes by the support driver, so a struct change forces a ring-0 rebuild
  and a `vboxdrv` reload (impossible while other VMs are live).
- Keeping the vestigial flash fully instantiated keeps the device's MMIO flash handle
  valid, so the *unmodified* ring-0 construct runs as-is. No ring-0 code changes.

The result is three small hunks in one file, with no ABI impact. See the
`org-ai-assisted/virtualbox` repo for the full modified `DevEFI.cpp` and the exact
diff recipe.

## Proof it works

Built and run on an OVH dedicated host (Debian trixie), 2026-10-21, with a 4 MiB
upstream coreboot `emulation/qemu-i440fx` image (`PAYLOAD_NONE`, serial console),
loaded by a non-hardened source build of VirtualBox 7.2.20 carrying the patch:

- VirtualBox log: `EFI: coreboot image, 0x400000 bytes mapped at 0xffc00000..4GB`.
- COM1 serial: coreboot ran `bootblock -> romstage -> postcar -> ramstage` and then
  enumerated VirtualBox's own emulated chipset:
  - `[8086/1237]` i440fx host bridge (VirtualBox PIIX3),
  - `[8086/7000]` PIIX3 ISA,
  - `[80ee/beef]` VirtualBox VGA,
  - `[80ee/cafe]` VirtualBox guest-service device,
  - `[8086/2415]` AC97 audio.

This is the decisive result: coreboot not only loads and executes under VirtualBox,
it correctly drives VirtualBox's virtual hardware. VirtualBox emulates the same
i440fx/PIIX3 chipset QEMU does, so upstream coreboot's QEMU board runs directly; the
one visible difference is that VirtualBox has no QEMU `fw_cfg` interface, so coreboot
takes its documented CMOS fallback for RAM sizing ("QEMU: firmware config interface
not found / Falling back to RAM info in CMOS"). With `PAYLOAD_NONE` coreboot stops
after device enumeration; a payload would boot on from there.

The full serial capture is in `org-ai-assisted/dasharo` under `proof/`.

## What was rejected

An earlier attempt merged VirtualBox's own `#ifdef VBOX` OVMF sources and Sovereign
Boot into a standalone `OvmfPkgX64` build. It compiled and loaded in VirtualBox but
hung early. That is the predictable outcome of merging two platform codebases, and it
is also the anti-pattern the edk2 project itself avoids: a new virtual platform gets
its own `OvmfPkg/<Platform>` DSC/FDF with runtime host-bridge detection, never an
`#ifdef <PLATFORM>` woven through shared files (the Bhyve port is the precedent).
VirtualBox's own firmware uses the `#ifdef VBOX` anti-pattern; we did not want to
extend it. The coreboot route sidesteps all of this: coreboot already has a clean
per-mainboard model.

## Clean next step: a real Dasharo target

`emulation/qemu-i440fx` proves the firmware path. The idiomatic Dasharo target is a
dedicated board rather than reusing the QEMU board:

1. A new coreboot mainboard `src/mainboard/emulation/qemu-vbox` (cloned from
   `qemu-i440fx`/`q35`, adapted to VirtualBox's CMOS RAM sizing and info port, with no
   `fw_cfg` dependency). coreboot's Bhyve support is the precedent for adding an
   emulator board without `#ifdef`-ing shared code.
2. The reused `DasharoPayloadPkg` as the coreboot payload, board-agnostic via the
   coreboot table.
3. A Dasharo `configs/config.emulation_vbox` wiring the two together.

That produces a genuine Dasharo firmware image for VirtualBox, loadable with the same
`DevEFI` `CorebootRom` path. Both changes (the coreboot board and the one-boolean
`DevEFI` loader) are small and self-contained enough to offer upstream.

## Decision

- Prefer real Dasharo (coreboot + `DasharoPayloadPkg`) over VirtualBox's OVMF fork for
  guests where the open firmware stack matters.
- The enabling VirtualBox change is the ring-3 `CorebootRom` loader -- minimal, no ABI
  impact, upstreamable -- not a merge of two edk2 platform trees.
- coreboot's per-mainboard model, not `#ifdef VBOX`, is the structure to extend for a
  first-class VirtualBox target.
