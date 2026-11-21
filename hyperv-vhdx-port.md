# A Hyper-V (VHDX) image target for Microsoft Windows native virtualization

## Status

Design and feasibility note, not yet built or booted. The firmware and conversion
findings below are read off the derivative-maker source and the qemu/Microsoft
documentation; the end-to-end Hyper-V boot has NOT been performed yet, so every
"boots" claim here is EXPECTED, not TESTED. The one empirical step that must close
before this is committed to is called out under "The one open verification". Scope of
the first pass is Kicksecure, single VM; the Whonix Gateway+Workstation pair is harder
and deferred (see "Whonix is harder").

## Problem

VirtualBox runs poorly on recent Windows. Windows 10/11 increasingly ship with
Virtualization Based Security (VBS), Hypervisor-Protected Code Integrity (HVCI), WSL2,
or Memory Integrity enabled by default, all of which turn on the Windows hypervisor
(Hyper-V). With Hyper-V present, VirtualBox cannot own the CPU's virtualization
extensions directly and falls back to a slow Hyper-V coexistence path (its NEM mode),
which is markedly slower and, on current builds, can fail outright. A Whonix user
reports exactly this: the Gateway and Workstation hang at a black screen during boot
under VirtualBox 7.2 on Windows 11 with VBS/HVCI/NEM active, and asks for a Hyper-V
variant
([forums.whonix.org/t/...23292](https://forums.whonix.org/t/whonix-vms-gateway-workstation-hang-at-black-screen-during-boot-under-virtualbox-7-2-10-on-windows-11-with-active-vbs-hvci-nem-mode/23292)).

The real fix is not a VirtualBox tweak on the user's host; it is to ship an image that
imports into Windows native virtualization (Hyper-V) directly. This note records what
that image is, how comparable projects build it, what derivative-maker already has, the
one genuinely new piece of work, and the recommended deliverable.

## How comparable projects ship a Hyper-V image

- Debian builds its Azure and cloud images with the FAI-based `debian-cloud-images`
  recipe (salsa `cloud/debian-cloud-images`); those VHDs are the de facto Debian
  Hyper-V/Azure artifact. Debian does not ship a general desktop Hyper-V image.
  Microsoft lists Debian 10-12 as supported on Hyper-V with UEFI and Secure Boot
  ([learn.microsoft.com supported-debian](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/supported-debian-virtual-machines-on-hyper-v)).
- Ubuntu ships desktop images in the Hyper-V Quick Create gallery, built with
  `livecd-rootfs` via a dedicated Hyper-V binary hook
  ([Launchpad #1837088](https://bugs.launchpad.net/bugs/1837088)) and pre-configured
  with xrdp for Enhanced Session Mode. Gallery delivery is a hosted `.vhdx` plus a JSON
  gallery manifest
  ([learn.microsoft.com custom-vm-gallery](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/custom-vm-gallery)).
- Linux Integration Services (the `hv_*` guest drivers) are mainline in the Linux
  kernel (`CONFIG_HYPERV*`), so a recent Debian kernel already carries them; you do NOT
  install Microsoft's downloadable LIS on Debian. The Debian `hyperv-daemons` package is
  userspace only (KVP/IP injection, VSS backup quiescing, file copy), not the drivers.
- Format: Hyper-V uses VHDX (modern; Gen2 boots from it) and legacy VHD (2 TB cap, no
  corruption resilience). VHDX is the correct target. Dynamic (grows on use) versus
  fixed (full-size file) is a download-size versus marginal-perf tradeoff; dynamic is
  the obvious choice for a downloadable image.

## Firmware: Gen2 works for free (the pivotal finding)

The widely repeated community blocker is that Kicksecure/Whonix ship only GRUB's BIOS
payload (`grub-pc`), so a Hyper-V Generation 2 (UEFI) VM cannot boot them; the community
guides walk through a manual `grub-pc` -> `grub-efi` plus GPT/ESP migration
([qua3k.github.io/guides/whonix](https://qua3k.github.io/guides/whonix)). That blocker
does NOT apply to a derivative-maker-built image. The amd64 raw image derivative-maker
produces is already hybrid BIOS + UEFI:

- grml-debootstrap builds the VM disk with a GPT label carrying an EFI System Partition
  (FAT32, `boot` flag), a `bios_grub` partition (GUID
  `21686148-6449-6e6f-744e-656564454649`), and the ext4 rootfs
  (`grml-debootstrap/grml-debootstrap`, the `--vmefi` path).
- Its `chroot-script` detects the `bios_grub` partition, selects
  `MAIN_GRUB_PACKAGE=grub-cloud-amd64` (plus `grub-efi-amd64-signed` and `shim-signed`),
  and installs grub BOTH ways: `grub-install --target=x86_64-efi
  --force-extra-removable` AND `grub-install --target=i386-pc`.

So one raw image, converted byte-for-byte to VHDX, is expected to boot:

- Hyper-V Gen1 (BIOS/MBR-style firmware) via the `i386-pc` grub, and
- Hyper-V Gen2 (UEFI/GPT) via the `x86_64-efi` grub, including Secure Boot via the
  Microsoft-signed `shim-signed` already in the image.

No repartitioning and no bootloader change are required. This is the payoff of reading
derivative-maker's actual build rather than trusting the generic community guides; the
boot-test lane already exercises this image under BIOS, EFI, and EFI + Secure Boot, so
the dual-firmware image is a known-good quantity (under qemu/OVMF; see the open
verification for the Hyper-V firmware specifically).

Secure Boot has one Hyper-V-side catch, documented for the import step, not the image:
the default "Microsoft Windows" Secure Boot template will not validate a Linux shim; the
VM must use the "Microsoft UEFI Certificate Authority" template
(`Set-VMFirmware -SecureBootTemplate MicrosoftUEFICertificateAuthority`), or Secure Boot
off
([learn.microsoft.com generation-2-security-features](https://learn.microsoft.com/windows-server/virtualization/hyper-v/generation-2-virtual-machine-security-features)).
Debian's shim is signed by Microsoft's third-party UEFI CA, so under that template
Secure Boot is expected to validate.

## The conversion: raw to VHDX with qemu-img

derivative-maker already owns the exact pattern. The qcow2 target
(`build-steps.d/4400_convert-raw-to-qcow2`) is a plain host-side
`qemu-img convert -O qcow2` from the finalized raw (`${binary_image_raw_file}`), with no
chroot and no proprietary tool. `qemu-utils` (which provides `qemu-img` with VHDX
support) is already a declared build dependency (`variables.d/60_dependencies.bsh`,
`help-steps/set-early-dependencies`). A VHDX target is a near-clone of that step:

```
qemu-img convert -p -f raw -O vhdx -o subformat=dynamic \
   "${binary_image_raw_file}" "${binary_image_vhdx_file}"
```

Two points are load-bearing:

- Sparse handling is the subtle one. Hyper-V refuses a disk file it considers compressed,
  encrypted, or sparse
  ([lists.gnu.org qemu-discuss 2019-10](https://lists.gnu.org/archive/html/qemu-discuss/2019-10/msg00023.html)),
  and `qemu-img` leaves zero runs as holes by default. The fix usually cited for this,
  `-S 0`, is the WRONG tool here: it writes every zero sector, fully allocating the
  output, so a `-S 0` dynamic VHDX of the build's default 100G virtual disk is ~100G on
  disk and to download even for a tiny guest, defeating `subformat=dynamic`
  ([qemu.org qemu-img](https://www.qemu.org/docs/master/tools/qemu-img.html)). The two
  sparsenesses are distinct: VHDX-internal dynamic block allocation (the BAT leaving
  unused blocks unallocated) is what we want and Hyper-V accepts it; what Hyper-V rejects
  is the host-side NTFS sparse FLAG on the `.vhdx` file on Windows. So the right shape is
  a genuinely compact dynamic VHDX whose delivered file carries no NTFS sparse attribute
  (cleared on Windows with `fsutil sparse setflag <file> 0`, or moot because we ship it
  compressed and extraction writes it dense) -- NOT `-S 0`. Which exact handling Hyper-V
  accepts is for the open verification below to settle empirically.
- Convert from the RAW we build, never from the VirtualBox `.ova` / VMDK. Converting a
  VMware/VirtualBox appliance's disk to VHDX is a documented Hyper-V boot-failure trap
  (fails on WS2022 in dynamic, fixed, and default subformats, Gen1 and Gen2; suspected
  appliance-specific boot records,
  [gitlab.com/qemu-project/qemu/-/issues/2204](https://gitlab.com/qemu-project/qemu/-/issues/2204)).
  The raw is clean and is what every existing target already converts from.

Note: Azure additionally wants a FIXED VHD with the virtual size aligned to a whole MB
(`force_size`, a `vpc`/VHD-only option, not available for `-O vhdx`). That is an
Azure-only constraint and does not apply to local Hyper-V import; we target local
Hyper-V, so dynamic VHDX is correct.

## The one genuinely new piece of work: reproducibility

derivative-maker's images are bit-for-bit reproducible, verified with diffoscope, and
the raw payload underneath a VHDX is already reproducible (`SOURCE_DATE_EPOCH`,
`FIXED_DISK_IDENTIFIERS`, pinned GPT part-UUIDs, the deterministic
`4350_reimage-raw-reproducible` pass, and `dm-normalize-fat-partition` for the ESP). The
only new nondeterminism a VHDX introduces is in the VHDX container header itself:
`qemu-img` stamps randomly generated GUIDs (the File Write GUID, the Data Write GUID,
and metadata-region GUIDs), a creation timestamp, and the two headers' sequence numbers
(QEMU seeds the first header's sequence number randomly and increments it for the second;
the header with the higher sequence number is the current one), none pinnable from the
CLI, so two builds would not be byte-identical and the diffoscope gate would redden.

The fix is a small post-convert normalizer modeled exactly on the existing
`packages/kicksecure/helper-scripts/usr/libexec/helper-scripts/vbox-ova-reproducible-normalize`,
which already rewrites VirtualBox's random disk/image/machine UUIDs and NIC MACs from
`SOURCE_DATE_EPOCH` plus a salt. A `vhdx-reproducible-normalize` would derive the VHDX
header GUIDs and both header sequence numbers deterministically from `SOURCE_DATE_EPOCH`
plus a salt (preserving which header is current), patch the header and metadata region,
and -- critically -- recompute each 4 KiB header's CRC-32C checksum after editing: a
header whose contents changed without a fresh checksum is invalid and the image is
corrupt. It then validates the normalized image still opens (`qemu-img info`, plus the
boot verification below) before the CI diffoscope gate covers reproducibility. (As in the
OVA normalizer, the UUID namespace constant must be generated by a human on a trusted
machine, not by an AI agent.) This is the only part that is new code rather than wiring.

## In-guest drivers

The `hv_*` drivers are mainline. The Kicksecure hardened-kernel config
(`packages/kicksecure/hardened-kernel/usr/share/hardened-kernel/hardened-host-kernel`)
builds `HYPERV_STORAGE` (hv_storvsc), `HYPERV_NET` (hv_netvsc), `HYPERV_KEYBOARD`,
`PCI_HYPERV`, and `FB_HYPERV` as modules, and the stock Debian kernels the default image
uses carry them too. The only in-guest concern is that the storage driver must be in the
initramfs so a Gen2 VM finds its root disk at first boot. Nothing in the tree currently
forces `hv_vmbus`/`hv_storvsc`/`hv_netvsc`/`hv_utils` into the initramfs; whether they
are already pulled in depends on the exact kernel config. The safe default is to add them
to `/etc/initramfs-tools/modules` and rebuild the initramfs, and optionally install
`hyperv-daemons` for integration services. This is confirmed only by booting (see below).

## Deliverable and import UX

There is no single-file OVA equivalent for Hyper-V
([learn.microsoft.com answers/843049](https://learn.microsoft.com/en-us/answers/questions/843049/ova-equivalent-for-hyper-v)).
Hyper-V Export/Import produces a folder (`.vhdx` plus a `.vmcx`/`.vmrs` config) that is
version-fragile across hosts and not a clean portable bundle. The least-friction path is
to ship the bare `.vhdx` (compressed, e.g. `.zip` or `.xz`) and have the user create a
new Gen2 VM attaching the existing disk.

Ship a short PowerShell import script alongside it; the script is what removes the two
footguns a GUI user gets wrong (selecting Generation 2, and the UEFI CA Secure Boot
template):

```powershell
New-VM -Name "Kicksecure" -Generation 2 -MemoryStartupBytes 2GB `
  -VHDPath "C:\path\Kicksecure.vhdx" -SwitchName "Default Switch"
Set-VMFirmware "Kicksecure" -SecureBootTemplate MicrosoftUEFICertificateAuthority
```

GUI step-by-step instructions are the fallback for users who will not run a script.

## Whonix is harder (deferred)

Kicksecure is a single VM and is the clean first target. Whonix is a Gateway +
Workstation pair that the VirtualBox `.ova` ships pre-wired on an internal network.
Because Hyper-V has no OVA, the Whonix deliverable additionally needs an internal/private
virtual switch created and both VMs attached to it, which is materially more work and
more boot-verification surface (two VMs, the internal-only network for the Workstation,
an external switch for the Gateway). The community Hyper-V Whonix guides confirm this
two-vSwitch shape
([forums.whonix.org/t/...6741](https://forums.whonix.org/t/how-to-get-whonix-running-on-hyper-v/6741),
[qua3k.github.io/guides/whonix](https://qua3k.github.io/guides/whonix)). Deferred to a
second pass once the single-VM Kicksecure path is proven.

## Implementation sketch (derivative-maker)

Reusing patterns already in the tree, scoped to one new target named `hyperv` (the name
`windows` is already taken: `--target windows` builds the VirtualBox Windows installer
`.exe`, Authenticode-verified, and is unrelated to a guest image):

1. New build step `build-steps.d/43xx_convert-raw-to-vhdx`, cloned from
   `4400_convert-raw-to-qcow2`, gated on a new `dist_build_hyperv`, running the
   `qemu-img ... -O vhdx` convert above from `${binary_image_raw_file}`.
2. New `vhdx-reproducible-normalize` under helper-scripts, cloned from
   `vbox-ova-reproducible-normalize`, pinning the VHDX header/metadata GUIDs and both
   header sequence numbers from `SOURCE_DATE_EPOCH` plus salt and recomputing each
   header's CRC-32C; invoked right after the convert.
3. Wiring: add `hyperv`/`dist_build_hyperv` to `help-steps/parse-cmd` (the target enum
   string, the `--help` list, the `--target` dispatch case, and the "at least one valid
   target" validation, the new boolean being `dist_build_hyperv`); add `binary_image_vhdx_file` to
   `variables.d/40_version-paths.bsh`; register the artifact for upload in
   `variables.d/50_upload.bsh`; add a `dist_build_hyperv` branch to `dm-prepare-release`'s
   `signatures_create` so the `.vhdx` is signed and uploaded; optionally add it to
   `help-steps/dm-build-official-one`'s per-arch default target set (otherwise reachable
   via `dist_build_multi_target_list=hyperv`).
4. Tests in `org-ai-assisted/dist-ai` mirroring the raw-to-iso and vbox suites (e.g. an
   unbound-var/static check on the new step, and a reproducibility assertion on the
   normalizer), plus a diffoscope gate in CI. Per project convention, comprehensive
   tests live in dist-ai, not in derivative-maker.
5. The PowerShell import script and user documentation ship as release collateral.

## The one open verification

Before committing to this design: build one UEFI-GPT Kicksecure raw image, convert it
with `qemu-img ... -O vhdx -o subformat=dynamic`, and boot it end to end in a real
Hyper-V Gen2 VM, both with Secure Boot on (UEFI CA template) and off. Confirm the root
disk mounts without a manual initramfs change on the exact target kernel
(`lsinitramfs | grep hv_storvsc`); if it does not, add the `hv_*` modules to the
initramfs as above. Also settle the sparse/allocation question empirically: confirm which
`.vhdx` shape Hyper-V accepts -- a compact dynamic VHDX with the Windows NTFS sparse flag
cleared -- without falling back to a fully-allocated `-S 0` image, and that the
reproducibility normalizer's output (pinned GUIDs and sequence numbers, recomputed
CRC-32C) still opens and boots. This closes the residual unknowns: that a Gen2 VM boots
our dual-firmware image, that the root disk is reachable out of the box, and that the
compact, reproducible VHDX imports cleanly. Only after that boots should the pipeline
work land.

## Decision (proposed)

- Add a first-class `hyperv` build target that emits a dynamic VHDX converted from the
  existing raw image (`qemu-img ... -O vhdx`, never from the `.ova`), reusing the
  qcow2 step as the template.
- Rely on the image's existing hybrid BIOS + UEFI boot, so one VHDX serves Hyper-V Gen1
  and Gen2 (Secure Boot capable via the signed shim) with no partition or bootloader
  change.
- Write one small `vhdx-reproducible-normalize` to keep the artifact bit-reproducible;
  that is the only genuinely new code.
- Ship the compressed `.vhdx` plus a PowerShell import script (Generation 2 and the
  Microsoft UEFI CA template baked in); there is no OVA-equivalent bundle to ship.
- Scope the first pass to Kicksecure (single VM). Defer the Whonix Gateway + Workstation
  pair (needs the internal/private vSwitch wiring) to a second pass.
- Gate the whole thing on the one open verification above: a real Hyper-V Gen2 boot,
  tested before the pipeline work is relied on.
