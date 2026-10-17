# Verified, measured, and IMA boot

How do you give a Debian-based distribution (Kicksecure/Whonix) verified boot,
measured boot, and an Integrity Measurement Architecture (IMA) path -- for both
the live ISO and the installed system -- while keeping GRUB, leaving the kernel
and initrd to Debian, keeping legacy BIOS as a fallback, and staying on a road
that can reach remote attestation later?

This note states the problem, records three facts that reframe it, surveys how
the major projects solve it (named, with citations), lays out the two viable
designs, reports what was empirically settled in a sandbox, and records the
recommended direction. It is the successor to the "measured/sealed boot chain"
limitation flagged at the end of [live-usb-persistence.md](live-usb-persistence.md).

Scope: the integrity MECHANISM (what verifies what, and how trust is anchored).
The read-only-root / overlay / state-separation machinery is covered where it
intersects the mechanism; the amnesic and persistence stories live in the
sibling note.


## Three facts that reframe the problem

1. **Debian trixie ships a shim-signed systemd-boot.** `systemd-boot-efi-amd64-signed`
   exists, analogous to `grub-efi-amd64-signed`. So "systemd-boot is not
   Microsoft/Debian-signed" is no longer a reason to avoid it. Preferring GRUB
   is now a feature decision (one loader for BIOS and UEFI, its scripting and
   its own GPG verifier), not a signing-gap decision. TESTED (see below).

2. **A Unified Kernel Image (UKI) does NOT take the kernel away from Debian.**
   `ukify`/`mkosi-initrd` wrap Debian's STOCK `vmlinuz` + STOCK initrd into one
   signed PE binary. You still run Debian's kernel; you never maintain a kernel.
   What a UKI takes over is bundling-and-signing the initrd + cmdline -- which
   must be done anyway (fact 3). So "leaves the kernel to Debian" is NOT a
   differentiator between the UKI and no-UKI designs; both ride Debian's kernel.

3. **There is no free verified boot.** Debian's stock `shim -> grub -> kernel`
   chain verifies the KERNEL (via the `shim_lock` verifier in GRUB's signed
   core) but NEVER the initrd. The initrd is generated locally at each kernel
   update, so Debian's private key is never present to sign it. Closing that
   gap REQUIRES owning a signing key and signing something (a UKI, or a detached
   signature) on every kernel/initrd update, whichever design is chosen. This
   unsigned-initrd gap is the whole reason the UKI exists.

The practical invariant that follows: whatever the design, Kicksecure will
(a) own an X.509 signing key, (b) enroll it via MOK into the `.machine` keyring
(stock trixie: `CONFIG_INTEGRITY_MACHINE_KEYRING=y`, `CONFIG_INTEGRITY_CA_MACHINE_KEYRING`
unset, so a plain leaf cert is accepted -- no CA, no kernel rebuild), and
(c) run a `postinst`/kernel/initramfs hook that re-signs on every update.


## How the major projects solve it

Convergence is strong, and it is the systemd model: a signed boot artifact
carries a Merkle-tree root digest for a read-only root on its signed command
line; measured boot goes into the TPM; IMA is reserved for attestation or
writable areas.

| Project | Loader | Read-only-root integrity | Roothash carrier (trust anchor) | Measured boot | IMA |
|---|---|---|---|---|---|
| systemd / ParticleOS | systemd-boot | dm-verity (/usr via `mount.usr=verity`) | signed UKI cmdline | PCR 11 (systemd-stub), sealed via systemd-pcrlock | no |
| Fedora bootc / sealed Atomic | systemd-boot (UKI) or shim->GRUB (ostree) | composefs + fs-verity (file-level) | fs-verity digest in signed UKI cmdline | PCR 11 / PCR 7 | no |
| openSUSE MicroOS / Aeon | systemd-boot | none (btrfs snapshots) | n/a -- integrity via TPM-sealed LUKS | PCR 0-7,11 via pcrlock / pcr-oracle | no |
| Ubuntu Core | grub (gadget) | dm-verity for snaps (new, 2.73+) | signed model assertion + kernel.efi | PCR 7, secboot ResealKeys | no |
| Arch / verity-squash-root | systemd-boot (UKI) | dm-verity over squashfs | signed UKI cmdline | PCR 11 | no |
| Android AVB | OEM/SoC bootloader | dm-verity | signed, rollback-protected vbmeta | vendor | no |
| ChromeOS | depthcharge | dm-verity | roothash on cmdline, signed in kernel-partition preamble | vendor | no |
| Flatcar | shim -> GRUB | dm-verity (/usr) | roothash in GRUB-verified signed load scripts | optional | no |
| Microsoft Azure Linux | GRUB | dm-verity | `verity.roothash=` on GRUB cmdline, signed boot chain | - | no |
| Talos | GRUB (BIOS) / systemd-boot+UKI (UEFI) | read-only squashfs in signed UKI | signed UKI | PCR 11 + 7 | no |
| Garden Linux | systemd-boot (UEFI only) | EROFS rootfs in signed UKI (rejected dm-verity) | signed UKI | PCR 11 | no |
| Vanilla OS 2 / ABRoot | GRUB (two-stage) | A/B OCI image (consistency, not verity) | n/a | - | no |

Three load-bearing observations:

- **No mainstream distro uses IMA/EVM as its ROOT integrity mechanism.** They
  all use dm-verity (block-level) or fs-verity/composefs (file-level). IMA's
  real-world footprint is remote attestation (Keylime) and enterprise/government
  hardened distros (RHEL, openEuler) that accept its operational cost. For a
  read-only verity root, IMA adds cost without benefit.
- **The deployed GRUB pattern is "sign the carrier," not "sign the roothash in
  the kernel keyring."** Azure Linux rides the signed boot chain, Flatcar signs
  the GRUB load scripts, ChromeOS signs the kernel partition. The in-kernel
  signed-roothash feature (`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG`) is used mainly
  where the carrier is NOT signed (Android post-boot volumes, DIY U-Boot/Yocto).
  No deployed distro combines GRUB + cmdline roothash + reliance on that kernel
  feature; that combination is experimental.
- **Every project that ships a full verified+measured path ships it UEFI-only.**
  Fedora sealed images, Ubuntu Core FDE, Talos SecureBoot -- all require UEFI
  Secure Boot + TPM 2.0. Legacy BIOS gets, at most, verity-at-rest with no
  trustworthy root of trust.


## The idiomatic 2025/2026 stack

1. A signed boot artifact (UKI) bundling kernel + initrd + cmdline, loaded
   behind shim.
2. A read-only root enforced by a Merkle tree (dm-verity, or fs-verity via
   composefs) whose root digest rides the signed artifact's command line, so
   the digest's trust derives from the artifact's signature.
3. Measured boot into PCR 11 (precomputable, vendor-signable), PCR 7 for Secure
   Boot state; TPM-sealed FDE resealed across updates by systemd-pcrlock.
4. A/B image updates (systemd-sysupdate / ostree / snaps); overlays via
   sysext/confext.
5. IMA optional -- attestation or writable-area coverage only.

The one genuine fork is composefs+fs-verity (Fedora) versus dm-verity (everyone
else). For a Debian base, dm-verity is the better fit: composefs tooling is
Fedora/ostree-centric and still experimental, whereas dm-verity over
squashfs/erofs is well-trodden in Debian live and in verity-squash-root.


## The two viable designs for Kicksecure

The choice is NOT "GRUB vs UKI" -- GRUB can CHAINLOAD a signed UKI. It is:

**Path A -- shim (Debian) -> Debian's signed GRUB -> your signed UKI.**
- Rides Debian's prebuilt signed shim + GRUB; no own GRUB build.
- `ukify` bundles Debian's stock kernel + initrd; you sign the UKI with your key
  (MOK/db enrolled). Initrd protection is BUNDLED (robust).
- dm-verity roothash on the UKI's signed cmdline (the "sign the carrier" model).
- Measured boot: systemd-stub measures the UKI into PCR 11 (deterministic).
- BIOS: the same GRUB loads extracted Debian kernel + initrd (a UKI cannot
  execute on BIOS); verity-at-rest only, no Secure Boot, no measured boot.
- Least new Kicksecure code. Matches Talos, Kairos, Garden Linux, Fedora sealed.

**Path B -- shim -> your OWN signed GRUB (GPG) -> Debian kernel + initrd.**
- You MUST build and sign your own GRUB `core.img` with the `pgp`+`verify`+`tpm`
  modules built in, a GRUB password + lockdown (to kill the `set
  check_signatures=no` console bypass), and your pubkey embedded (the pubkey
  must live in the shim-verified PE, so Debian's prebuilt GRUB cannot carry it).
- GRUB enforces `check_signatures=enforce` over detached signatures of kernel,
  initrd, and grub.cfg -- the GRUB-native way to close the initrd gap without a
  UKI. Historically unreliable as a distro default (Debian bug #906124: an image
  with `linux` but not `pgp` is a bypass) and exposed to the console bypass.
- dm-verity roothash on the signed grub.cfg cmdline.
- Measured boot: GRUB's `tpm` module into PCR 8/9 (noisy, non-precomputable).
- Matches Flatcar, Azure Linux, ChromeOS (the carrier-is-signed half).

`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG` is DEFENSE-IN-DEPTH in either path, never
the anchor: a tampered initrd can simply not activate verity and pivot
elsewhere, so the kernel-keyring check only prevents a forged-but-"verified"
rootfs substitution, not "the system actually uses the signed root." The anchor
is whatever signs the initrd + cmdline.


## Empirically settled (sandbox, Debian trixie)

A real UEFI + Secure Boot + TPM 2.0 harness was built inside the sandbox qube
(OVMF secboot firmware with Microsoft-key variables, swtpm 2.0 via `tpm-crb`,
qemu TCG, every boot timeout-bounded). Two facts were settled; both labeled
TESTED.

- **Signed Debian GRUB embeds and enforces the `tpm` module and measures PCR
  8/9 under Secure Boot: YES.** Boot chain `MS shim -> Debian-signed grub ->
  kernel` ran with Secure Boot ON (proven by the PCR 7 event-log entry
  `SecureBoot="01"`). `tpm2_eventlog` on the kernel-captured
  `binary_bios_measurements` showed PCR 8 events (`grub_cmd: insmod tpm`,
  `kernel_cmdline: ...`) and PCR 9 events (`grub.cfg`, `/vmlinuz`, initrd). The
  `tpm` module is in the signed core (`CD_MODULES` in Debian's
  `debian/build-efi-images`; `strings` on `grubx64.efi.signed` shows
  `grub_tpm_measure`). This resolves the old pkg-grub-devel ambiguity
  affirmatively for trixie.

- **`shim -> Debian signed grub -> your MOK/db-signed UKI` boots end-to-end
  under Secure Boot with PCR 11 measured: YES.** A `ukify`-built UKI (Debian
  stock vmlinuz + initrd + cmdline) signed with an own key enrolled in the UEFI
  `db` chainloaded cleanly; the kernel inside started
  (`KERNELVER=6.12.111+deb13-amd64`); systemd-stub measured all UKI sections
  (`.linux`, `.osrel`, `.cmdline`, `.initrd`, `.uname`, `.sbat`) into PCR 11.
  Negative control: one flipped byte in the signed UKI -> `error: bad shim
  signature`, kernel never started. Verification is real, not bypassed.

Versions tested: grub-efi-amd64-signed 1+2.12+9+deb13u2, shim-signed
1.51~1+deb13u1+16.1-2~deb13u1, systemd/ukify/systemd-boot-efi 257.13-1~deb13u1,
linux-image 6.12.111+deb13-amd64 (TPM driver built-in; stock config has
`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG=y`, full IMA/EVM, the IPE LSM).

Confidence caveats: TCG not KVM (semantically identical for SB/TPM, timing
only); trust was established via the UEFI `db` route (a sanctioned equivalent),
so the interactive MokManager UI path was not separately exercised (high
confidence it behaves identically). Debian's signed-grub prefix is `/EFI/debian`.

Net: Path A is validated on stock Debian trixie, not merely plausible.


## Measured boot for remote attestation

Remote attestation tolerates GRUB's "non-precomputable PCR 8/9" weakness, which
is only a problem for LOCAL sealing (binding a secret to a golden PCR). In
attestation the verifier REPLAYS the event log against the TPM quote after the
fact, so unpredictable-but-logged PCRs are fine:

1. `TPM2_Quote` returns a signed structure over selected PCRs + a verifier
   nonce, signed by an Attestation Key tied to the Endorsement Key.
2. The logs explain the opaque PCRs: the UEFI TCG2 event log (PCRs 0-7), GRUB's
   PCR 8/9, the IMA log (PCR 10).
3. The verifier checks the quote signature and nonce, replays each log entry
   (`PCR = H(PCR || digest)`), compares to the quoted values, then applies
   policy to the now-trusted log contents.

Keylime implements exactly this (quote over PCRs + `binary_bios_measurements` +
the IMA log; a measured-boot policy and a runtime IMA policy). GRUB on EFI
extends PCR 8/9 through the firmware's `EFI_TCG2_PROTOCOL HashLogExtendEvent`,
so a verifier sees firmware + Secure Boot + GRUB in one continuous log.

Path A gives the cleaner PCR 11 as well (better reference-state stability than
PCR 8/9). Either way the attestation MODEL is sound; the roughness in the GRUB
PCR 8/9 path is operational (brittle reference state), not theoretical.


## IMA / EVM

- **Measurement-only (`ima_policy=tcb` -> PCR 10) is deployable on stock Debian
  today.** One kernel cmdline flag populates the measurement log and extends PCR
  10; no xattrs, no signing, no keyring, never blocks execution. This is the
  attestation path and the place to start. SUSE's Keylime guide uses exactly
  this.
- **Appraisal-enforce (`ima_appraise=enforce`) + EVM is a genuine heavy lift,
  blocked by a real Debian gap.** It needs a valid `security.ima` xattr on every
  file; dpkg has NO per-file signing (RPM/Fedora/SUSE do), so `apt` writes
  unlabeled files that then fail appraisal. Workable only on a frozen,
  self-signed appliance image, not a rolling desktop. Scope it narrowly; do not
  put it on the general-purpose system until Debian gains package file signing.


## Installed system: verity root + working apt

The design must cover the installed system, not only the live ISO. The tension:
a dm-verity root is read-only and immutable (a single signed root hash), but apt
wants to write `/usr`, `/etc`, `/var`, and the dpkg database. Every shipping
image-based system resolves this by moving the write OUT of the running system.
Three patterns:

- (a) **A/B image swap** (Vanilla OS/ABRoot, Ubuntu Core, bootc): never `apt
  install` into the live root; build/pull a new signed image and reboot.
  Strongest atomicity, weakest "it is still apt" feel.
- (b) **Immutable /usr + writable /etc,/var overlay** (systemd
  `mount.usr=verity`, sysext/confext): the vendor tree is sealed, machine state
  is writable. Clean integrity story, but no on-device package manager (sysext
  is not apt: additive, no dependency resolution).
- (c) **Reboot-gated mutable escape hatch, then re-seal** (openSUSE MicroOS
  `transactional-update`; Kicksecure's own documented sysmaint design): keep the
  REAL package manager, mutate in a dedicated maintenance boot, regenerate +
  re-sign the verity image afterward.

Kicksecure's documented user/sysmaint boot-mode design IS pattern (c): normal
(user) boots are verity read-only; a SYSMAINT boot mounts the root writable,
runs real apt, and regenerates + re-signs on shutdown, keeping a backup copy for
rollback. It maps onto MicroOS's transactional-update UX, not ABRoot's A/B OCI
swap. No Debian port of transactional-update exists, so the apt-preserving
transactional workflow is greenfield on the Debian base.

Engineering guidance:

- Reuse systemd's verity + roothash + `systemd-repart` plumbing (as `mkosi`
  assembles it; it treats Debian/Ubuntu as first-class) as the SEALING layer,
  and build only the Kicksecure-specific reseal/re-sign on top -- do not reinvent
  the verity/UKI plumbing.
- ONE verity-over-squashfs/erofs image can back BOTH targets: the live ISO
  (writes -> tmpfs overlay, `systemd.volatile=overlay`) and the installed disk
  (writes -> a persistent `repart`-created LUKS `/var`+`/etc`). Only the
  writable-layer wiring differs; the sealed image is shared. See
  verity-squash-root and KDE Linux's verity scheme.
- Close one real gap in the documented design: the current single-backup-copy
  rollback is weaker than an A/B verity slot. Adding a second verity slot is
  cheap (only the sealed image duplicates, not a full ABRoot A/B root) and
  removes the brick risk without compromising the "apt still works" property.


## BIOS: a degraded fallback, by design

BIOS is a fallback and does not need the full guarantee; worst case, duplicate
the kernel/initrd. The relevant facts:

- **isolinux/syslinux cannot read squashfs at all.** GRUB2's i386-pc (legacy
  BIOS) target CAN: it boots a kernel out of a squashfs on SeaBIOS using the
  stock Debian `grub-pc-bin` `squash4` + `loopback` + `iso9660` modules (TESTED
  end-to-end). So keeping squashfs on BIOS means using GRUB, not isolinux.
- **Verified boot on BIOS is partial at best.** GRUB-pc can load kernel+initrd
  with `roothash=` on the cmdline and the kernel's dm-verity enforces root-fs
  integrity at REST. But BIOS runs the MBR with NO signature check, so
  `core.img`, `grub.cfg`, kernel and initrd are all unverified; a medium rewrite
  defeats it. A UKI cannot execute on BIOS.
- **Measured boot on BIOS is effectively unavailable:** commodity legacy BIOS
  has no trustworthy SRTM, no standardized TCG event log (so pcrlock/pcr-oracle
  cannot work), no Secure Boot state in PCR 7.

So BIOS is documented as an explicitly reduced-integrity path: verity-at-rest
only, no Secure Boot, no measured boot, not marketed as "verified boot."


## Recommended direction

Adopt **Path A** (shim -> Debian signed GRUB -> your signed UKI) on UEFI, with
GRUB loading extracted Debian kernel+initrd as the degraded BIOS entry. It keeps
GRUB (one loader for both firmwares), leaves the kernel+initrd to Debian
(`ukify` only bundles them), needs no own GRUB build, closes the initrd gap
robustly (bundled), and gives deterministic PCR 11 -- the better substrate for
the remote-attestation goal. It is empirically validated on stock trixie.

Path B (own signed GRUB, GPG-enforced detached signatures, no UKI) is the right
choice ONLY if a UKI artifact is refused on principle and the cost of owning a
GRUB build is accepted; it matches Flatcar/Azure and is legitimate, just more
new code and a weaker initrd story.

Start IMA at measurement-only (`ima_policy=tcb` + Keylime); defer appraisal+EVM
to a frozen-appliance scope until Debian gains package-level file signing.


## Open product decisions

- **Path A (GRUB -> signed UKI) vs Path B (GRUB-GPG, no UKI anywhere).** Given
  "UKI not required but GRUB preferred," Path A delivers both and is recommended
  unless a hard "no UKI artifact" rule applies.
- **Rollback: keep the single backup copy, or move to A/B verity slots?**
  (A/B recommended.)
- **IMA appraisal: ever, or measurement-only indefinitely?** Gated on Debian's
  missing per-file `.deb` signing.


## Relationship to the Kicksecure wiki

The Kicksecure wiki pages Verified_Boot, Measured_boot, Sovereign_Boot,
Dev/Stateless, Dev/Immutable, and Grub-live describe the goals and a partial
design, but as of this writing nothing is implemented, the Measured_boot page is
an empty stub, IMA/EVM is absent, and the live-ISO path is unaddressed. Sovereign
Boot (the user-controlled Secure Boot key-enrollment wizard, 3mdeb/Dasharo) is
the key-ownership front-end for the MOK enrollment this design needs. Today's
`grub-live` (dracut `90overlay-root` + `rootovl` + `live-hardener`) is
statelessness, explicitly NOT an integrity boundary -- the gap this design
closes.


## Citations

- systemd trusted boot: https://0pointer.net/blog/brave-new-trusted-boot-world.html ; https://0pointer.net/blog/fitting-everything-together.html
- UKI spec: https://uapi-group.org/specifications/specs/unified_kernel_image/ ; systemd-stub: https://man7.org/linux/man-pages/man7/systemd-stub.7.html
- TPM2 PCR measurements: https://systemd.io/TPM2_PCR_MEASUREMENTS/ ; PCR registry: https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/
- systemd-pcrlock: https://man7.org/linux/man-pages/man8/systemd-pcrlock.8.html ; systemd-measure: https://man7.org/linux/man-pages/man1/systemd-measure.1.html
- ParticleOS: https://github.com/systemd/particleos ; mkosi: https://github.com/systemd/mkosi
- Discoverable Partitions Spec: https://uapi-group.org/specifications/specs/discoverable_partitions_specification/
- composefs: https://github.com/composefs/composefs ; ostree composefs: https://ostreedev.github.io/ostree/composefs/
- Fedora sealed Atomic: https://fedoramagazine.org/sealed-atomic-desktops-test-images/ ; bootc: https://bootc.dev/
- openSUSE systemd-boot FDE: https://microos.opensuse.org/blog/2023-12-20-sdboot-fde/ ; pcr-oracle retirement: https://news.opensuse.org/2026/03/11/dropping-pcr-oracle/
- transactional-update: https://github.com/openSUSE/transactional-update
- Ubuntu Core FDE: https://documentation.ubuntu.com/core/explanation/full-disk-encryption/ ; secboot: https://github.com/snapcore/secboot
- Arch dm-verity + UKI, verity-squash-root: https://github.com/brandsimon/verity-squash-root/ ; KDE Linux verity: https://community.kde.org/KDE_Linux/Verity
- sbctl: https://github.com/Foxboron/sbctl ; systemd rootfs discovery: https://systemd.io/ROOTFS_DISCOVERY/
- Android AVB: https://source.android.com/docs/security/features/verifiedboot/dm-verity ; ChromeOS: https://www.chromium.org/chromium-os/chromiumos-design-docs/verified-boot/
- Flatcar secure boot: https://hackmd.io/@flatcar/H1MDJuu7lg ; Azure Linux read-only roots: https://mintlify.wiki/microsoft/azurelinux/security/read-only-roots
- Talos SecureBoot: https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/bare-metal-platforms/secureboot ; Kairos trusted boot: https://kairos.io/docs/architecture/trustedboot/ ; Garden Linux boot modes: https://github.com/gardenlinux/gardenlinux/blob/main/docs/explanation/boot-modes.md
- Vanilla OS ABRoot: https://github.com/Vanilla-OS/ABRoot
- GRUB digital signatures: https://www.gnu.org/software/grub/manual/grub/html_node/Using-digital-signatures.html ; measured boot: https://www.gnu.org/software/grub/manual/html_node/Measured-Boot.html ; Debian grub bug #906124: https://alioth-lists.debian.net/pipermail/pkg-grub-devel/2019-July/016723.html ; grub tpm-in-signed-core thread: https://lists.debian.org/pkg-grub-devel/2021/08/msg00039.html
- Debian grub signed-image modules: https://salsa.debian.org/grub-team/grub/-/raw/master/debian/build-efi-images ; Debian SecureBoot: https://wiki.debian.org/SecureBoot
- dm-verity (kernel): https://docs.kernel.org/admin-guide/device-mapper/verity.html
- IMA concepts/policy: https://ima-doc.readthedocs.io/en/latest/ima-concepts.html ; https://ima-doc.readthedocs.io/en/latest/ima-policy.html ; Keylime: https://keylime.dev ; https://keylime.readthedocs.io/en/latest/user_guide/use_measured_boot.html
