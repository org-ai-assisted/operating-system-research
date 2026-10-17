# Verified, measured, and IMA boot

How do you give a Debian-based distribution (Kicksecure/Whonix) verified boot,
measured boot, and an Integrity Measurement Architecture (IMA) path -- for both
the live ISO and the installed system -- while keeping GRUB, leaving the kernel
and initrd to Debian, keeping legacy BIOS as a fallback, and staying on a road
that can reach remote attestation later?

This note states the problem, records three facts that reframe it, surveys how
the major projects solve it (named, with citations), lays out the viable
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
   `ukify` wraps Debian's STOCK `vmlinuz` and (via `--initrd=/boot/initrd.img-*`)
   Debian's STOCK initrd into one signed PE binary. (`mkosi-initrd` instead
   builds a FRESH initrd, not the stock one, so it is not the way to keep
   Debian's initramfs.) You still run Debian's kernel; you never maintain a kernel.
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
(a) own an X.509 signing key; (b) enroll it via MOK. A normal `mokutil --import`
(a leaf cert, no module-signing-only KeyUsage OID) makes shim trust that key to
validate the bootloader, the UKI AND the kernel it loads -- the full boot chain,
not just modules. To ALSO anchor a dm-verity roothash signature in-kernel, the
owner must additionally run `mokutil --trust-mok`, which admits the key to the
trusted `.machine` keyring (linked into `.secondary`); trixie's shipped kernel
already has the compile-time gates (`CONFIG_INTEGRITY_MACHINE_KEYRING=y`,
`CONFIG_INTEGRITY_CA_MACHINE_KEYRING` unset so a leaf cert is accepted,
`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG` + `..._SECONDARY_KEYRING=y`, read from
`/boot/config`), so only the `--trust-mok` runtime step is needed -- no kernel
rebuild. And (c) run a `postinst`/kernel/initramfs hook that re-signs on every
update.


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
| Microsoft Azure Linux | GRUB | dm-verity | roothash on GRUB cmdline (Mariner `rd.verityroot.roothash=`), signed boot chain | - | no |
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


## The viable designs for Kicksecure

The requirement is subtler than it first looks: verified boot needs boot
CONFINED to the signed artifact, not merely that the signed artifact verifies
when it is chosen. An empirical test (below) confirmed that `shim -> Debian's
signed GRUB -> a signed UKI` chainload verifies and refuses a tampered UKI. That
is necessary but NOT sufficient: **Debian's stock signed GRUB cannot confine
boot to the UKI.** Its `grubx64.efi` has `linux`/`initrd` built in, ships no
`pgp`/`verify` module, and reads an UNSIGNED `/EFI/debian/grub.cfg`; `shim_lock`
validates only the kernel PE. So an attacker (or anyone at the `e` menu)
rewrites grub.cfg to `linux` the Debian-signed kernel with an attacker-controlled
initrd -- Secure Boot still on, the UKI never executed. The signed-roothash
kernel feature does not save this: a hostile initrd simply never activates
verity and pivots elsewhere. So "ride Debian's prebuilt GRUB and chainload a UKI"
is NOT a verified-boot config. That leaves two real designs plus one non-answer:

**Option 1 (idiomatic, recommended) -- shim -> Debian's signed systemd-boot ->
your signed UKI.** systemd-boot has no bare-kernel/`initrd` command: it loads
only PE binaries (UKIs), each shim-verified, so boot IS confined to signed UKIs.
Debian trixie now ships a shim-signed systemd-boot (fact 1), so this needs no
own-build and the original reason to prefer GRUB is moot. `ukify` bundles
Debian's stock kernel+initrd; you sign the UKI with your MOK-enrolled key; the
dm-verity roothash rides the UKI's signed cmdline; systemd-stub measures PCR 11
(deterministic). GRUB stays only as the degraded BIOS loader. This is exactly how
Talos, Kairos, Garden Linux and Fedora sealed images load their UKIs -- via
systemd-boot, NOT GRUB.

**Option 2 (keeps GRUB on UEFI) -- shim -> your OWN signed GRUB (GPG) -> Debian
kernel + initrd.** The ONLY way to keep GRUB as the UEFI loader AND close the
gap: build+sign your own `grubx64.efi` with `pgp`+`verify`+`tpm` built in, your
pubkey embedded (it must live in the shim-verified PE -- Debian's prebuilt GRUB
cannot carry it), a GRUB password, and `check_signatures=enforce` over
grub.cfg+kernel+initrd, so grub.cfg and the initrd are finally signature-checked.
dm-verity roothash on the signed grub.cfg cmdline. Historically fragile (Debian
bug #906124: a core with `linux` but not `pgp` is a bypass) and the console `set
check_signatures=no` must be locked out. Measured boot via GRUB's `tpm` module
into PCR 8/9 (noisy, non-precomputable). Matches Flatcar/Azure/ChromeOS
(carrier-is-signed). Cost: you own a GRUB build + key.

**Non-answer -- shim -> Debian's signed GRUB -> signed systemd-boot/UKI.**
Tempting, because it keeps Debian's prebuilt GRUB as the first stage. It does
NOT close the gap: the same unsigned grub.cfg + built-in `linux`/`initrd` let an
attacker skip systemd-boot and boot the signed kernel with an evil initrd.
Keeping Debian's stock GRUB anywhere in front of the kernel reintroduces the hole
unless grub.cfg is itself signed -- which is Option 2. Not a real third option.

`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG` is DEFENSE-IN-DEPTH in either option,
never the anchor: a tampered initrd can simply not activate verity and pivot
elsewhere, so the kernel-keyring check only prevents a forged-but-"verified"
rootfs substitution, not "the system actually uses the signed root." The anchor
is whatever CONFINES boot and signs the initrd+cmdline -- systemd-boot+UKI
(Option 1) or own-GRUB+`check_signatures` (Option 2).


## Secure Boot key ownership: MS-shim compatibility and Sovereign Boot

The default key strategy and Sovereign Boot live at different layers and are
selected by the owner, so Kicksecure can offer BOTH:

- **Default -- MS-signed shim + additive MOK.** Debian's shim is
  Microsoft-signed, so it boots on the large installed base of SB-enabled
  machines that trust the MS keys, and shim ALSO consults the MOK list. The owner
  enrolls their own key with `mokutil --import` WITHOUT removing any Microsoft or
  Debian key -- purely additive. This is the light form of owner control and the
  mode the designs above assume (the MOK key signs the UKI / own GRUB).
- **Sovereign Boot (opt-in, heavy) -- take ownership.** The 3mdeb/Dasharo
  provisioning wizard is the full feudal-to-sovereign step: it forces Setup Mode,
  DELETES the platform key `PK`, wipes KEK/db, pushes the Microsoft certificates
  into `dbx` (so nothing MS-signed boots), enrolls only the owner's keys into db,
  and re-takes ownership with a fresh ephemeral `PK`. The owner's key becomes the
  sole trust root.

They conflict only at the strict end, and the resolution is a spectrum, not one
config: ship MS-shim-compatible by default (works everywhere; additive MOK for
owners who want to sign their own UKIs/kernels), and present full Sovereign Boot
as a deliberate owner-driven migration. Full sovereignty has real costs -- the
MS-signed shim itself lands in `dbx` and becomes unusable, so the whole chain
must be signed by the owner's key, and dual-boot / vendor recovery /
firmware-capsule paths that rely on MS signatures break. It also sidesteps the
mid-2026 Microsoft 2011-CA expiry (the replacement UEFI CA 2023 must reach each
machine's db via firmware/OS updates; an owner who roots trust only in their own
db does not depend on that rotation). Status: Sovereign Boot is RC/demo-stage
(3mdeb spec v0.1.0), not a shipped default.


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
  signature`, kernel never started. This proves signature enforcement on the UKI
  PATH is real -- but NOT that boot is confined to that path: with Debian's stock
  GRUB the unsigned grub.cfg still lets an attacker bypass the UKI and boot the
  Debian-signed kernel with an evil initrd (see "The viable designs"). The test
  validates the chosen path, not path confinement.

Versions tested: grub-efi-amd64-signed 1+2.12+9+deb13u2, shim-signed
1.51~1+deb13u1+16.1-2~deb13u1, systemd/ukify/systemd-boot-efi 257.13-1~deb13u1,
linux-image 6.12.111+deb13-amd64 (TPM driver built-in; stock config has
`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG=y`, full IMA/EVM, the IPE LSM).

Confidence caveats: TCG not KVM (semantically identical for SB/TPM, timing
only); trust was established via the UEFI `db` route (a sanctioned equivalent),
so the interactive MokManager UI path was not separately exercised (high
confidence it behaves identically). Debian's signed-grub prefix is `/EFI/debian`.

Net: signature enforcement on the UKI path, and GRUB's PCR 8/9 measurement, are
validated on stock trixie. Path CONFINEMENT is the separate requirement the
design section resolves -- it needs systemd-boot (Option 1) or an own-signed
GRUB (Option 2), because stock Debian GRUB does not provide it.


## Measured boot for remote attestation

Remote attestation tolerates GRUB's "non-precomputable PCR 8/9" weakness, which
is only a problem for LOCAL sealing (binding a secret to a golden PCR). In
attestation the verifier REPLAYS the event log against the TPM quote after the
fact, so unpredictable-but-logged PCRs are fine:

1. At ENROLLMENT the verifier binds an Attestation Key (AK) to the TPM's
   Endorsement Key (EK) via credential activation (`TPM2_MakeCredential` /
   `ActivateCredential`). A `TPM2_Quote` signature ALONE does not prove the AK
   belongs to a real TPM, so this binding step is mandatory -- omit it and a
   software key can forge a quote.
2. `TPM2_Quote` returns a structure over selected PCRs + a verifier nonce,
   signed by the enrolled AK.
3. The logs explain the opaque PCRs: the UEFI TCG2 event log (PCRs 0-7), GRUB's
   PCR 8/9, the IMA log (PCR 10).
4. The verifier checks the quote signature (against the enrolled AK) and nonce,
   then REPLAYS the logs: for each event it recomputes the digest FROM the event
   body, folds `PCR = H(PCR || digest)` in order (skipping non-extended
   `EV_NO_ACTION` records such as the Spec ID event), and checks the result
   equals the quoted PCRs. Recomputing from the body is what catches a tampered
   event; only then is the log content trusted.

Keylime implements exactly this (quote over PCRs + `binary_bios_measurements` +
the IMA log; a measured-boot policy and a runtime IMA policy). GRUB on EFI
extends PCR 8/9 through the firmware's `EFI_TCG2_PROTOCOL HashLogExtendEvent`,
so a verifier sees firmware + Secure Boot + GRUB in one continuous log.

Option 1 (systemd-boot + UKI) gives the cleaner PCR 11 as well (better
reference-state stability than PCR 8/9). Either way the attestation MODEL is
sound; the roughness in the GRUB PCR 8/9 path is operational (brittle reference
state), not theoretical.

### Prior art on a Debian base (as of 2026-10): greenfield

Remote attestation on a Debian base barely exists. Keylime -- the reference
TPM-measured-boot + IMA remote-attestation stack -- is NOT packaged in Debian or
Ubuntu (upstream supports only RHEL 9+ and SUSE SLE Micro; the one Ubuntu PPA is
a dead 2021 personal build), and no Debian-derived security distro (Kicksecure,
Whonix, Tails, PureOS) ships or documents it. The one genuinely deployed
Debian-base case is cloud-tied: Ubuntu confidential VMs on Azure (vTPM + AMD
SEV-SNP, attested to Microsoft Azure Attestation; GA for SEV-SNP, TDX in
preview). Ubuntu Core/Desktop TPM-backed FDE does measured boot but only LOCAL
key sealing, not verifier attestation. The mature OS-integrated comparator is
SUSE SLE Micro, which ships Keylime (IMA + measured boot to a verifier) as a
documented feature. So the attestation goal means packaging/integrating Keylime
(or equivalent) ourselves -- there is no Debian-native path to inherit.


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
- **The gap is acknowledged but unresolved in the Debian BTS.** debhelper
  #766267 (install `security.ima` xattrs at unpack) and apt #823945 (parse PAX
  headers / restore xattrs on extraction, with a working patch) are both OPEN
  and never merged: apt deferred to dpkg, which never implemented xattr/IMA
  support. The effort was IBM-IMA-developer-driven (~2014-2016), not Debian
  maintainers; there is no DEP. RPM solved the same problem with
  `rpm-plugin-ima` (file signatures in the RPM header, `security.ima` written at
  install) -- Debian has no equivalent, which is exactly why appraisal is
  impractical on a rolling Debian system.
- **A dpkg hook could retrofit `security.ima`, but only as local-hash
  integrity.** A `DPkg::Post-Invoke` hook walking `dpkg-query -L` and calling
  `evmctl` (from the packaged `ima-evm-utils`) can set `security.ima` on
  dpkg-owned files -- mechanically feasible on stock Debian, and the only path
  given the missing native support. But it yields integrity / offline-tamper-at-
  rest only (a local hash, or a local/TPM-sealed key), NOT vendor authenticity:
  Debian's build infra signs no file contents, so the hook certifies "what dpkg
  unpacked on this box," not "what Debian built and signed" -- a package
  compromised at/before install is faithfully signed. It also covers only the
  dpkg-owned subset: every non-dpkg path (generated config, logs, runtime data,
  the locally built initramfs) must be excluded or path-whitelisted by the IMA
  policy, which is where the real operational cost and breakage risk live. EVM
  (`security.evm`, key ideally TPM-sealed) must protect the xattrs. Defensible
  as a defense-in-depth at-rest layer beside dm-verity + FDE; not a substitute
  for provenance, and not to be advertised as vendor-verified. Nobody ships this
  on stock Debian.


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
- **Measured boot on BIOS is effectively unavailable:** the TCG PC Client
  firmware-log format does cover conventional BIOS, but commodity legacy BIOS
  rarely implements a trustworthy measured-boot SRTM, exposes no reliable event
  log, and has no Secure Boot state in PCR 7 -- so pcrlock/pcr-oracle
  precomputation and a trustworthy root of trust are absent in practice.

So BIOS is documented as an explicitly reduced-integrity path: verity-at-rest
only, no Secure Boot, no measured boot, not marketed as "verified boot."


## Recommended direction

Adopt **Option 1** -- `shim -> Debian's signed systemd-boot -> your signed UKI`
on UEFI, with GRUB kept only as the degraded BIOS loader. It confines boot to
signed UKIs (systemd-boot has no bare-kernel command), rides Debian's now-signed
systemd-boot (no own-build), leaves the kernel+initrd to Debian (`ukify` only
bundles them), bundles+signs the initrd, puts the dm-verity roothash on the UKI's
signed cmdline, and gives deterministic PCR 11 -- the better substrate for the
remote-attestation goal. It is how the comparable projects (Talos, Kairos, Garden
Linux, Fedora sealed) actually load UKIs.

Choose **Option 2** (own signed GRUB + `check_signatures`) only if keeping GRUB
as the UEFI loader is a hard requirement -- it is the sole way to do that AND
close the initrd gap, at the cost of owning a GRUB build + key and a noisier
PCR 8/9 measured-boot story.

Do NOT keep Debian's stock GRUB in front of the kernel on the verified path: its
unsigned grub.cfg + built-in `linux`/`initrd` reintroduce the gap regardless of
what it chainloads. (The earlier draft's "ride Debian's stock GRUB -> UKI" is
withdrawn for exactly this reason.)

Start IMA at measurement-only (`ima_policy=tcb` + Keylime); defer appraisal+EVM,
or a local-hash dpkg hook, to a frozen-appliance scope until Debian gains
package-level file signing.


## Open product decisions

- **Option 1 (systemd-boot + UKI) vs Option 2 (own signed GRUB).** systemd-boot
  is recommended now that Debian signs it; choose Option 2 only if GRUB on the
  UEFI path is mandatory. Note this is a change from an earlier draft that
  recommended riding Debian's stock GRUB to chainload a UKI -- that does not
  confine boot and was withdrawn.
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
- Ubuntu Core FDE: https://documentation.ubuntu.com/core/explanation/full-disk-encryption/ ; secboot: https://github.com/canonical/secboot
- Arch dm-verity + UKI, verity-squash-root: https://github.com/brandsimon/verity-squash-root/ ; KDE Linux verity: https://community.kde.org/KDE_Linux/Verity
- sbctl: https://github.com/Foxboron/sbctl ; systemd rootfs discovery: https://systemd.io/ROOTFS_DISCOVERY/
- Android AVB: https://source.android.com/docs/security/features/verifiedboot/dm-verity ; ChromeOS: https://www.chromium.org/chromium-os/chromiumos-design-docs/verified-boot/
- Flatcar secure boot: https://hackmd.io/@flatcar/H1MDJuu7lg ; Azure Linux read-only roots: https://mintlify.wiki/microsoft/azurelinux/security/read-only-roots
- Talos SecureBoot: https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/bare-metal-platforms/secureboot ; Kairos trusted boot: https://kairos.io/docs/architecture/trustedboot/ ; Garden Linux boot modes: https://github.com/gardenlinux/gardenlinux/blob/main/docs/explanation/boot-modes.md
- Vanilla OS ABRoot: https://github.com/Vanilla-OS/ABRoot
- GRUB digital signatures: https://www.gnu.org/software/grub/manual/html_node/Using-digital-signatures.html ; measured boot: https://www.gnu.org/software/grub/manual/html_node/Measured-Boot.html ; Debian grub bug #906124: https://alioth-lists.debian.net/pipermail/pkg-grub-devel/2019-July/016723.html ; grub tpm-in-signed-core thread (Bug#846383): https://alioth-lists.debian.net/pipermail/pkg-grub-devel/2021-August/017634.html
- Debian grub signed-image modules: https://salsa.debian.org/grub-team/grub/-/raw/master/debian/build-efi-images ; Debian SecureBoot: https://wiki.debian.org/SecureBoot
- dm-verity (kernel): https://docs.kernel.org/admin-guide/device-mapper/verity.html
- IMA concepts/policy: https://ima-doc.readthedocs.io/en/latest/ima-concepts.html ; https://ima-doc.readthedocs.io/en/latest/ima-policy.html ; Keylime: https://keylime.dev ; https://keylime.readthedocs.io/en/latest/user_guide/use_measured_boot.html
- Debian IMA/EVM per-file-signing gap (acknowledged, unresolved): debhelper bug #766267 https://bugs.debian.org/766267 ; apt bug #823945 https://bugs.debian.org/823945 ; apt-devel patch thread https://lists.debian.org/deity/2016/05/msg00034.html ; Keylime install (RHEL/SUSE only) https://keylime.readthedocs.io/en/latest/installation.html
- Debian-base remote attestation prior art: Ubuntu confidential VMs on Azure https://ubuntu.com/blog/ephemeral-ubuntu-confidential-vms-azure ; Azure guest attestation https://learn.microsoft.com/azure/confidential-computing/guest-attestation-confidential-vms ; SUSE SLE Micro attestation (mature comparator) https://documentation.suse.com/sle-micro/5.5/html/SLE-Micro-all/cha-security-attestation.html
- dpkg per-file signing / DebSignatures (draft, stalled): https://wiki.debian.org/Teams/Dpkg/Spec/DebSignatures ; Fedora rpm-plugin-ima (the model Debian lacks): https://packages.fedoraproject.org/pkgs/rpm/rpm-plugin-ima/ ; ima-evm-utils (evmctl): https://packages.debian.org/sid/ima-evm-utils
- MOK scope (shim validates bootloader+kernel+modules; .machine keyring for dm-verity): shim README https://github.com/rhboot/shim ; rodsbooks controlling Secure Boot https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html ; kernel module signing https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html ; Ubuntu Secure Boot (MOK, module-signing-only KeyUsage OID) https://documentation.ubuntu.com/security/docs/security-features/platform-protections/secure-boot/
- Sovereign Boot: Kicksecure wiki https://www.kicksecure.com/wiki/Sovereign_Boot ; Dasharo wizard https://docs.dasharo.com/projects/sovereign-boot-wizard/ ; 3mdeb verified-boot app spec https://github.com/3mdeb/verified-boot/blob/master/verified_boot_app_spec.md ; Microsoft 2011 CA expiry / UEFI CA 2023 https://eclypsium.com/blog/microsoft-secure-boot-certificates-expire-2026/
