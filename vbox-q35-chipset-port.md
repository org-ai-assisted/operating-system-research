# Porting a Q35 chipset to VirtualBox for coreboot/Dasharo firmware

## Problem

[dasharo-virtualbox-port.md](dasharo-virtualbox-port.md) showed that a flat
`coreboot.rom` can boot in VirtualBox via a ring-3 `DevEFI` raw-loader change, and
that coreboot can drive VirtualBox's existing `piix3` (i440fx) chipset. Its "clean
next step" was a first-class emulation target. This note records the next step
actually taken: porting a Q35/ICH9 chipset into VirtualBox so coreboot's upstream
`emulation/qemu-q35` board -- the board Dasharo's UEFI + Sovereign-Boot features are
built on -- boots, without breaking VirtualBox's default firmware.

VirtualBox ships only two chipsets: `piix3` (i440fx, legacy, no PCIe ECAM, no
fw_cfg) and `ich9`. Its `ich9` is an i440fx-class host bridge with an ECAM window
bolted on: it exposes MMCONFIG + an ACPI MCFG, but it is not a real Q35 -- no Q35
MCH host bridge (the D0:F0 device is `#if 0`'d out), no programmable `PCIEXBAR`, no
SMM, and no ICH9 SMBus controller. coreboot's `qemu-q35` board expects a real-enough
Q35: it programs `PCIEXBAR` and halts (`mainboard_machine_check`) if the readback is
wrong, and `i82801ix` early-init calls `enable_smbus()` which dies if no Intel SMBus
device answers. So `qemu-q35` cannot boot on stock VirtualBox `ich9`.

## How the reference (QEMU) models Q35

coreboot's `qemu-q35` is written against QEMU's `q35` machine. The relevant
hardware contracts, from the QEMU/Q35 and ICH9 datasheet behaviour coreboot relies
on:

- **MCH host bridge (D0:F0), `PCIEXBAR` (0x60, 64-bit).** Firmware writes the ECAM
  base + length + enable bit; the MCH relocates the MMCONFIG window to that base.
  coreboot's `make_pciexbar()` uses `CONFIG_ECAM_MMCONF_BASE_ADDRESS` (default
  `0xb0000000`, 256 buses), writes it via legacy CF8/CFC, then validates the
  readback. QEMU honours the relocation; real hardware does too.
- **ICH9 LPC (D31:F0), `PMBASE` (0x40) + `ACPI_CNTL` (0x44).** Firmware programs the
  ACPI power-management I/O block base; the LPC relocates the PM1/PM-timer/GPE I/O
  decode there. coreboot `i82801ix` sets `DEFAULT_PMBASE` and advertises the PM
  timer at `PMBASE + 8` in the FADT. QEMU's ICH9 honours the relocation.
- **ICH9 SMBus (D31:F3), Intel vendor 0x8086.** `enable_smbus()` finds the device
  by (fixed) devfn, reads its vendor id, sets the SMB I/O base + enable, and resets
  the host controller. On an emulated board it then does no real SPD transaction.
- **SMM** is optional in coreboot (`CPU_QEMU_X86_NO_SMM` selects `NO_SMM`).

## What VirtualBox actually provides (and the gaps)

Verified empirically and in the VirtualBox 7.2.20 source (`DevPciIch9.cpp`,
`DevACPI.cpp`, `ConsoleImplConfigX86.cpp`):

- `ich9` exposes a real ECAM window (`McfgBase` ~ `0xdc000000` = 4G - RAM-hole, 64
  buses) + an ACPI MCFG -- but the base is hypervisor-fixed, not firmware-programmable
  (no MCH, no `PCIEXBAR`).
- No ICH9 SMBus controller at D31:F3.
- The ACPI PM block is modelled by VirtualBox's own `DevACPI` PCI device:
  `acpiR3CalcPmPort() = uPmIoPortBase + offset`, default `PM_PORT_BASE = 0x4000`.
  `uPmIoPortBase` IS relocatable at runtime -- but only via a write to **that ACPI PCI
  device's** `PMBA` register (`acpiR3PciConfigWrite -> acpiR3UpdatePmHandlers`). The
  **ICH9 LPC `PMBASE`** register is NOT wired to it. So on the i440fx/PIIX4 path a
  guest relocating PM works; on the ich9 path it does not.

## The port (VirtualBox side, opt-in, legacy-safe)

An opt-in `Q35Chipset` mode was added to `DevPciIch9.cpp`
(`org-ai-assisted/virtualbox-coreboot@ai`), CFGM key
`VBoxInternal/Devices/ich9pci/0/Config/Q35Chipset`, **default off** so legacy `ich9`
is byte-for-byte unchanged (verified: stock EFI still boots the desktop with the
rebuilt device library). It adds the two devices coreboot `qemu-q35` needs:

- **Q35 MCH host bridge at 00:00.0** (Intel 0x8086 / 0x29c0). A config-access
  intercept treats a `PCIEXBAR` (reg 0x60/0x64) write like a real MCH: it relocates
  VirtualBox's ECAM MMIO window to the firmware-programmed base (cf. QEMU's
  `mch_update_pciexbar`). So `machine_check` passes and ECAM lands where the q35
  board expects it (`0xb0000000`).
- **ICH9 SMBus stub at 00:1f.3** (Intel 0x8086 / 0x2930, SMBus class). Presence-only:
  `enable_smbus()` only checks the vendor id and writes config; qemu-q35 issues no
  real SMBus transaction, so no I/O backend is needed. (This required raising the
  root device's `cMaxPciDevices` to 2, an R0-visible DEVREG change.)

## The coreboot side

Build `emulation/qemu-q35` with `CONFIG_CPU_QEMU_X86_NO_SMM=y` (VirtualBox has no
SMRAM/TSEG), keeping `UefiPayloadPkg`. One VirtualBox-specific workaround is needed:

- **`DEFAULT_PMBASE` 0x600 -> 0x4000** for the VirtualBox target. Because the ICH9
  LPC `PMBASE` write is not wired to VirtualBox's `DevACPI` PM block (above), the PM
  block stays at `0x4000`. coreboot's default `0x600` would put the FADT PM timer at
  `0x608`, which VirtualBox does not decode. Setting `PMBASE = 0x4000` aligns the
  generated FADT PM timer (`0x4008`) with VirtualBox's live timer.

The PMBASE gap surfaced as a subtle hang, worth recording: the edk2 `UefiPayloadPkg`
binds `AcpiTimerLib` (the non-universal payload), so `MicroSecondDelay` -- used by
`CpuDxe`'s MP-init AP wait -- polls the FADT ACPI PM-timer I/O port. With the FADT
pointing at the dead `0x608`, the delay spun forever and the BSP wedged right after
setting up the AP wakeup vector (identical code sailed through on i440fx, whose PM
block IS relocated). Pointing `PMBASE` at `0x4000` fixed it. Separately, build the
payload as **RELEASE**: a DEBUG payload halts at a benign `DXE_ASSERT` in
`DxeTcgPhysicalPresenceLib` at BdsDxe (no TPM present); RELEASE skips asserts and
boots.

## Result (verified)

With `Q35Chipset` on and the coreboot q35 ROM, VirtualBox boots bootblock ->
romstage (past `machine_check` and `enable_smbus`) -> ramstage ("QEMU Northbridge
q35", ECAM at `0xb0000000`) -> ACPI/SMBIOS/coreboot tables -> `UefiPayload` ->
DxeCore -> BDS -> and the full Kicksecure LXQt desktop renders (1024x768, ~8815
distinct colours, same as the i440fx path). Stock VirtualBox EFI (Q35Chipset off)
still boots the desktop with the rebuilt device library -- the default firmware is
unbroken.

Test-environment gotcha worth recording: a self-built VirtualBox and the packaged
VirtualBox share `/dev/vboxdrv`, and supdrv caches ring-0 modules by name. If a
packaged-VirtualBox VM is running, the self-built userspace silently REUSES the
stock ring-0 modules, so an R0-visible change (the `cMaxPciDevices` bump) fails
`VERR_INCOMPATIBLE_CONFIG`. Loading the modified R0 requires stopping every
packaged-VirtualBox VM and building the own ring-0 set (`VMMR0` + `VBoxDDR0`). All
VirtualBox VMs on the test host run as root (vboxdrv's open handler EPERMs non-root).

## What was rejected

- **Upgrading stock `ich9` in place** (no opt-in flag): rejected -- it would change
  legacy `ich9` behaviour and risk the VirtualBox-upstream contribution. The gate is
  off by default; legacy stays identical.
- **Adopting coreboot's `qemu-q35` wholesale onto the existing VirtualBox ECAM**
  without an MCH: rejected -- `qemu-q35` builds and validates `PCIEXBAR` and dies on a
  wrong readback, so a programmable MCH is mandatory, not optional.
- **Making the SMBus stub functional** (real I/O backend): rejected as unnecessary --
  qemu-q35 never issues a transaction; a presence stub satisfies `enable_smbus()`.

## Decision

The Q35 port works and is separable for independent VirtualBox-upstream
contribution (it is distinct from the `DevEFI` CorebootRom loader and the ACPI
SLP_EN patches, and gated off by default). Both paths now boot the full Kicksecure
desktop on VirtualBox: `i440fx`/`piix3` (the simpler path, no chipset change) and
`q35`/`ich9` (`Q35Chipset` on, for the Dasharo-shaped firmware).

The one firmware workaround -- coreboot `PMBASE = 0x4000` -- should ideally move to
the VirtualBox side: wire the ICH9 LPC `PMBASE`/`ACPI_CNTL` write to
`acpiR3UpdatePmHandlers` so VirtualBox relocates its ACPI PM block like the PIIX4
path, after which coreboot's standard `0x600` works unmodified. That is the clean
next step.

Code: `org-ai-assisted/virtualbox-coreboot` (DevPciIch9 Q35Chipset, DevEFI
CorebootRom, DevACPI SLP_EN), `org-ai-assisted/coreboot` (qemu-q35 build + bochs
VBox-VGA + i82801ix PMBASE), `org-ai-assisted/edk2` (UefiPayloadPkg fb-retag PCD).
Operational specifics (build/boot harness, the ring-0 gotcha) live in dist-ai
`usr/share/dasharo-vbox-boot-tests/AGENTS.md`, not here.
