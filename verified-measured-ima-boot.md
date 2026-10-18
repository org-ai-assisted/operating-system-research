# Verified, measured, and IMA boot

How do you give a Debian-based distribution (Kicksecure/Whonix) verified boot,
measured boot, and an Integrity Measurement Architecture (IMA) path to a
PREVENTION standard (Sovereign / Secure Boot -- BLOCK a tampered boot, not merely
detect it) -- for both the live ISO and the installed system -- while keeping
MOK functional, leaving the kernel and initrd to Debian, keeping legacy BIOS as a
fallback, and staying on a road that can reach remote attestation later?

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
not just modules. That same MOK key also anchors a dm-verity roothash signature in-kernel: it is
admitted to the trusted `.machine` keyring (linked into `.secondary`). On DEBIAN
this is automatic -- Debian carries `trust-machine-keyring-by-default.patch`
(`trust_moklist()` returns true unconditionally), so a plain `mokutil --import`
leaf cert lands in `.machine` with NO `mokutil --trust-mok` needed (that extra
step is an upstream/vanilla requirement, not Debian's). Trixie's shipped kernel
has the compile-time gates (`CONFIG_INTEGRITY_MACHINE_KEYRING=y`,
`CONFIG_INTEGRITY_CA_MACHINE_KEYRING` unset so a leaf cert is accepted,
`CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG` + `..._SECONDARY_KEYRING=y`, read from
`/boot/config`) -- no kernel rebuild. And (c) run a `postinst`/kernel/initramfs hook that re-signs on every
update. (This MOK path is a SHIM feature and applies to the default, MS-shim
design. Full Sovereign Boot drops shim, so there is no MOK at all -- trust then
roots in your own keys in UEFI `db`, which the kernel loads directly into its
`.platform` keyring, and dm-verity anchors via `..._PLATFORM_KEYRING` instead of
MOK/`.machine`. See the Sovereign Boot section.)


Sources: Debian SecureBoot https://wiki.debian.org/SecureBoot ; Debian trust-machine-keyring-by-default patch https://sources.debian.org/src/linux/6.12.107-1/debian/patches/features/all/db-mok-keyring/trust-machine-keyring-by-default.patch

## How the major projects solve it

Convergence is strong, and it is the systemd model: a signed boot artifact
carries a Merkle-tree root digest for a read-only root on its signed command
line; measured boot goes into the TPM; IMA is reserved for attestation or
writable areas.

| Project | Loader | Read-only-root integrity | Roothash carrier (trust anchor) | Measured boot | IMA |
|---|---|---|---|---|---|
| systemd / ParticleOS | systemd-boot | dm-verity on /usr (`mount.usr=dissect` + image policy `usr=signed`) | signed UKI image policy (roothash auto-discovered from partition UUIDs) | PCR 11 (systemd-stub), signed PCR-11 policy (`SignExpectedPcr`), not pcrlock | no |
| Fedora bootc / sealed Atomic | systemd-boot (UKI) or shim->GRUB (ostree) | composefs + fs-verity (file-level) | fs-verity digest in signed UKI cmdline | PCR 11 / PCR 7 | no |
| openSUSE MicroOS / Aeon | systemd-boot | none (btrfs snapshots) | n/a -- integrity via TPM-sealed LUKS | PCR 0,2,4,7,9 (pcr-oracle; migrating to pcrlock, 2026-03) | no |
| Ubuntu Core | grub (gadget) | dm-verity for essential snaps (snapd 2.73+) | signed model assertion + kernel.efi | PCR 7, secboot ResealKeys | no |
| Arch / verity-squash-root | systemd-boot (UKI) | dm-verity over squashfs | signed UKI cmdline | - (SB + dm-verity; PCR 11 only implicit, not documented) | no |
| Android AVB | OEM/SoC bootloader | dm-verity | signed, rollback-protected vbmeta | vendor | no |
| ChromeOS | depthcharge | dm-verity | roothash on cmdline, signed in kernel-partition preamble | vendor | no |
| Flatcar | shim -> GRUB | dm-verity (/usr) | roothash in GRUB-verified signed load scripts | optional | no |
| Microsoft Azure Linux | GRUB | dm-verity | roothash on GRUB cmdline (Mariner `rd.verityroot.roothash=`; Azure Linux 3.0 Image Customizer `verity.roothash=`), signed boot chain | - | no |
| Talos | GRUB (BIOS) / systemd-boot+UKI (UEFI) | read-only squashfs in signed UKI | signed UKI | PCR 11 + 7 | no |
| Garden Linux | systemd-boot (UEFI only) | EROFS rootfs in signed UKI (chose over dm-verity) | signed UKI | PCR 7 (UKI sections also measured into 11) | no |
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


Sources: ParticleOS https://github.com/systemd/particleos ; mkosi https://github.com/systemd/mkosi ; composefs https://github.com/composefs/composefs ; ostree composefs https://ostreedev.github.io/ostree/composefs/ ; Fedora sealed Atomic https://fedoramagazine.org/sealed-atomic-desktops-test-images/ ; bootc https://bootc.dev/ ; openSUSE systemd-boot FDE https://microos.opensuse.org/blog/2023-12-20-sdboot-fde/ ; pcr-oracle retirement https://news.opensuse.org/2026/03/11/dropping-pcr-oracle/ ; Ubuntu Core FDE https://documentation.ubuntu.com/core/explanation/full-disk-encryption/ ; secboot https://github.com/canonical/secboot ; verity-squash-root https://github.com/brandsimon/verity-squash-root/ ; KDE Linux verity https://community.kde.org/KDE_Linux/Verity ; Android AVB https://source.android.com/docs/security/features/verifiedboot/dm-verity ; ChromeOS https://www.chromium.org/chromium-os/chromiumos-design-docs/verified-boot/ ; Flatcar https://hackmd.io/@flatcar/H1MDJuu7lg ; Azure Linux read-only roots https://mintlify.wiki/microsoft/azurelinux/security/read-only-roots ; Talos SecureBoot https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/bare-metal-platforms/secureboot ; Kairos trusted boot https://kairos.io/docs/architecture/trustedboot/ ; Garden Linux boot modes https://github.com/gardenlinux/gardenlinux/blob/main/docs/explanation/boot-modes.md ; Vanilla OS ABRoot https://github.com/Vanilla-OS/ABRoot

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


Sources: systemd trusted boot https://0pointer.net/blog/brave-new-trusted-boot-world.html ; https://0pointer.net/blog/fitting-everything-together.html ; UKI spec https://uapi-group.org/specifications/specs/unified_kernel_image/ ; systemd-stub https://man7.org/linux/man-pages/man7/systemd-stub.7.html ; systemd-pcrlock https://man7.org/linux/man-pages/man8/systemd-pcrlock.8.html ; systemd-measure https://man7.org/linux/man-pages/man1/systemd-measure.1.html ; Discoverable Partitions Spec https://uapi-group.org/specifications/specs/discoverable_partitions_specification/ ; sbctl https://github.com/Foxboron/sbctl ; systemd rootfs discovery https://systemd.io/ROOTFS_DISCOVERY/

## The viable designs for Kicksecure

Verified boot needs the TAMPERED boot to be PREVENTED or at least DETECTED. The
first instinct -- make the bootloader refuse anything but a signed UKI -- does
NOT work on a Debian base, and it is worth stating plainly why:

**No bootloader confines boot while Debian's (or Microsoft's) keys are trusted.**
- Debian's stock GRUB has `linux`/`initrd` built in and reads an UNSIGNED
  `grub.cfg`. It DOES embed the `pgp`/`verify` module (`check_signatures`,
  `verify_detached`, `trust` -- verified by `strings` on the trixie signed
  `grubx64.efi`), but ships NO embedded trusted key and does not force
  `check_signatures`, so there is no trust anchor by default. Rewrite grub.cfg
  (or use the `e` menu) to `linux` the Debian-signed kernel with an evil initrd;
  `shim_lock` checks only the kernel PE.
- systemd-boot is no better. It also loads Boot Loader Specification **Type #1**
  entries (`linux` + a SEPARATE initrd), and per systemd's own `UEFI_SECURITY.md`
  "initrds are also not verified in BLS Type #1 configurations." Drop a
  `loader/entries/evil.conf` and it boots the signed kernel + unsigned initrd.
  (So "systemd-boot only loads UKIs" is false -- an earlier draft of this note
  claimed it; it is withdrawn.)
- And the SAME Debian key signs BOTH systemd-boot and `grubx64.efi`, so an
  attacker who can write the ESP swaps the shim second-stage for whichever signed
  loader gives the easiest unsigned-initrd path. shim accepts it, Secure Boot
  stays on, the UKI never runs.

So while the firmware trusts a kernel (Debian/MS-signed) that accepts an unsigned
external initrd, and the ESP/boot-config is unauthenticated, the evil-initrd boot
cannot be PREVENTED by choosing a loader. `CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG`
does not save it either: a hostile initrd simply never activates verity. A signed
UKI makes the INTENDED boot signed and measurable; it does not stop an attacker
choosing a different path. That leaves two real anchors, meant to be used
together:

**Anchor 1 (the mainstream guarantee) -- measured boot + TPM-sealing /
attestation.** The intended boot (`shim -> systemd-boot -> signed UKI`) measures
the UKI into PCR 11; any tampered path (a Type #1 entry, a swapped GRUB, an evil
initrd) measures DIFFERENT PCRs. Bind the LUKS/FDE key to the expected PCRs with
`systemd-cryptenroll` + `systemd-pcrlock` (seal to PCR 7 +11, not the volatile
8/9): a tampered boot then cannot obtain the TPM-SEALED key, and remote
attestation of those PCRs FAILS. Caveat: `cryptenroll` ADDS a TPM keyslot and
leaves the existing passphrase keyslot, so an evil initrd that simply prompts for
the passphrase still unlocks the disk unless you `--wipe-slot=password` (TPM-only
unlock, with an offline recovery key kept). So this is DETECTION (a failed
auto-unlock / unexpected prompt is the signal) plus protection of the auto-unlock
secret -- it WITHHOLDS the sealed key, it does not by itself refuse the boot or
stop passphrase capture. This is what Ubuntu Core and the systemd model deliver. Practically: ship signed UKIs (so PCR 11 is
deterministic and vendor-signable), avoid shipping Type #1 entries, and
understand the guarantee comes from the TPM, not the loader.

**Anchor 2 (the only loader-level PREVENTION) -- Sovereign Boot / own key
hierarchy.** Remove Debian/Microsoft trust (enroll only your own keys, push the
MS certs into `dbx`) and sign ONLY your own UKIs -- never a kernel that takes a
separate initrd, never a GRUB with an unsigned cfg. Now no trusted artifact
accepts an unsigned initrd, so the evil-initrd path is genuinely cut. This is the
Sovereign Boot direction (next section); it buys PREVENTION rather than mere
DETECTION, at the cost of out-of-the-box compatibility.

An own-signed GRUB with `check_signatures=enforce` over grub.cfg+kernel+initrd
closes the GRUB layer. The `pgp`/`verify` module is ALREADY in Debian's grub;
what the own core adds is YOUR public key embedded as the trust anchor with
enforcement forced -- and you cannot inject `--pubkey`/forced-enforce into
Debian's PREBUILT SIGNED core without re-signing it (hence own core + a GRUB
password). But while the Debian key is still trusted an attacker swaps in
Debian's signed systemd-boot or GRUB, so it only PREVENTS once Debian trust is
also removed, i.e. it collapses into Anchor 2.
On its own it adds nothing over Anchor 1, and it still costs an own GRUB build.

The practical shape for Kicksecure: Anchor 1 by default (keeps MS-shim
compatibility, works out of the box, detects tamper), with Sovereign Boot
(Anchor 2) as the opt-in step up to prevention. A signed UKI is the carrier in
both; the roothash rides its signed cmdline; `ukify` keeps the kernel+initrd
Debian's.

Sources: GRUB digital signatures https://www.gnu.org/software/grub/manual/grub/html_node/Using-GPG_002dstyle-digital-signatures.html ; GRUB measured boot https://www.gnu.org/software/grub/manual/html_node/Measured-Boot.html ; Debian grub bug #906124 https://alioth-lists.debian.net/pipermail/pkg-grub-devel/2019-July/016723.html ; tpm-in-signed-core thread Bug#846383 https://alioth-lists.debian.net/pipermail/pkg-grub-devel/2021-August/017634.html ; dm-verity (kernel) https://docs.kernel.org/admin-guide/device-mapper/verity.html

### Signing the initrd without a UKI

A UKI is one carrier for the signed initrd, not the only one -- but the
alternatives are narrow. The kernel NEVER verifies a standalone initrd (the EFI
initrd-load path runs after ExitBootServices unchecked; the 2013 in-kernel
external-initramfs-signature RFC was rejected as insecure and never merged), and
systemd-boot does not verify Type #1 (`linux`+`initrd`) entries either -- which
is exactly why the UKI exists. The ONLY component that refuses to boot a tampered
standalone initrd is GRUB with `check_signatures=enforce` verifying a detached
GPG signature -- and for that to be non-bypassable you need an OWN-SIGNED GRUB
core with YOUR public key embedded as the trust anchor and enforcement forced.
(Debian's stock signed `grubx64.efi` DOES contain the `pgp`/`verify` module, but
with no embedded trusted key and `check_signatures` unset by default, and you
cannot inject `--pubkey`/forced-enforce into the prebuilt signed core without
re-signing it -- so you build your own.) So "sign the initrd instead of a UKI" is
not a shortcut: it IS the own-GRUB design, and it still needs your own key plus a
re-sign-on-regeneration hook (`/etc/initramfs/post-update.d/`, which fires after
every initramfs rebuild).

The UKI-vs-bare-initrd choice is thus about WHAT is signed and WHO verifies, not
about escaping key management:
- UKI: kernel+initrd+cmdline in one PE, one signature, verified by systemd-stub;
  cleaner precomputable PCR 11.
- Bare signed initrd: only the initrd is signed, verified by own-GRUB; the
  kernel+cmdline stay on the shim->grub->kernel chain with the unsigned-grub.cfg
  caveat, so the command line is NOT integrity-protected the way a UKI's embedded
  cmdline is -- unless grub.cfg is signed too (which own-GRUB `check_signatures`
  does).

For DETECTION/attestation no signing is needed at all: GRUB's `tpm` module
already measures the initrd into PCR 9 (and commands + cmdline into PCR 8), so a
tampered initrd is caught by attestation after the fact. PREVENTION is harder
than "own-GRUB" alone suggests: own-GRUB + `check_signatures` closes the GRUB
LAYER, but while Debian's shim (which embeds the Debian CA -- see the
key-ownership section) still launches stock Debian-signed GRUB, an attacker
simply swaps your own-GRUB for Debian's and regains the unsigned-grub.cfg +
evil-initrd path. So loader-level prevention additionally requires an OWN shim
(owner `VENDOR_CERT`) or revoking Debian's CA -- it is NOT achieved by signing
the initrd/GRUB while keeping Debian's shim. For Kicksecure's
keep-GRUB/keep-shim/MOK stack the realistic posture is DETECTION: measured boot
for attestation (PCR 8/9/10/11) plus TPM-sealed FDE. Seal to PCR 7 (+11 for a
UKI) via a signed `pcrlock` prediction, NOT to the non-precomputable PCR 8/9
(which would refuse to unseal on the next kernel/initrd/grub.cfg update), and
always keep an enrolled offline recovery key. ("Zero new SIGNING keys" is true;
sealing still adds a TPM-bound keyslot, and leaving a passphrase keyslot means an
evil initrd can still capture the passphrase -- so this is detection, not
prevention; see Anchor 1.)

Practicality note: GRUB `check_signatures` is essentially UNDOCUMENTED for Debian
users. The only real how-to is the upstream GNU GRUB manual; Debian ships the
tooling with no procedural docs, and real write-ups are rare and skew to Gentoo
(own UEFI `db` key) or Libreboot/Canoeboot (own firmware), not stock Debian.
Combining it with Debian's Secure Boot chain is effectively undocumented and
structurally awkward (you must build+sign+enroll your own GRUB core), and Debian
bug #906124 shows that path regressed AND raised a Secure-Boot-bypass hazard
(nothing guarantees `pgp` is loaded whenever `linux` is). So the own-GRUB route
is genuinely new ground: Kicksecure would have to provide the key-gen, the signed
self-built core, the signing hooks, enrollment, and a mandatory GRUB password
itself.


Sources: standalone initrd not kernel-verified, 2013 RFC https://lkml.iu.edu/hypermail/linux/kernel/1302.0/02432.html ; GRUB GPG-style digital signatures https://www.gnu.org/software/grub/manual/grub/html_node/Using-GPG_002dstyle-digital-signatures.html ; Fedora grub security modules https://fedoraproject.org/wiki/Changes/Include_security_modules_in_efi_Grub2 ; systemd-stub/UKI rationale https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html ; Debian re-sign hook points https://kernel-team.pages.debian.net/kernel-handbook/ch-update-hooks.html ; Gentoo Secure Boot/GRUB https://wiki.gentoo.org/wiki/Secure_Boot/GRUB ; Libreboot/Canoeboot GRUB hardening https://canoeboot.org/docs/gnulinux/grub_hardening.html ; Debian bug #906124 https://bugs.debian.org/906124

### Changing the kernel command line under a signed UKI

A signed UKI fixes the command line (its signed `.cmdline` section), so "edit the
boot entry" does NOT work under Secure Boot. The legitimate ways to change it, in
order of how well they preserve the signature guarantee:

1. **Rebuild + re-sign the UKI with the owner's key (canonical).** Edit the source
   cmdline (`kernel-install` reads it from the first of `/etc/kernel/cmdline`,
   `/usr/lib/kernel/cmdline`, `/proc/cmdline`), regenerate with
   `ukify --cmdline=...`, re-sign with `--secureboot-private-key` /
   `--secureboot-certificate`. Preserves the guarantee fully -- the key holder is
   the trust root. This is the same postinst/kernel-hook re-sign Kicksecure runs
   anyway (fact 3); `/etc/kernel/cmdline` is the Debian edit point.
2. **Signed UKI addons (append without rebuilding).** `*.addon.efi` PE files carry
   an extra `.cmdline` (built by `ukify build --cmdline=...` in addon mode, then
   PE-signed). systemd-stub loads global addons from `/loader/addons/*.addon.efi`
   and per-UKI addons from `<uki>.efi.extra.d/*.addon.efi` and -- verbatim -- "In
   case Secure Boot is enabled, these files will be validated using keys in UEFI
   DB, Shim's DB or Shim's MOK, and only loaded if the check passes." An unsigned
   addon is rejected under SB, so the guarantee holds. Concatenation order: UKI
   cmdline first, then global addons, then per-UKI addons; all measured into PCR 12.
3. **Boot-menu editing is BLOCKED (not merely discouraged).** The sd-boot `e`
   editor passes a cmdline as EFI invocation parameters, but systemd-stub's
   `settle_command_line()` DROPS it when `secure_boot_enabled() && (a .cmdline
   section is present || confidential VM)` (VERIFIED in `src/boot/stub.c`). So
   editing silently has NO effect for a signed UKI under SB; an override needs
   either SB off or a UKI shipped WITHOUT a `.cmdline` section.
4. **SMBIOS Type 11 (`io.systemd.stub.kernel-cmdline-extra`) -- works under SB but
   is UNAUTHENTICATED; do not trust it.** Contrary to a common assumption, the stub
   appends this host/VMM-supplied string even with Secure Boot ON:
   `cmdline_append_and_measure_smbios()` is gated ONLY on `is_confidential_vm()`,
   NOT on Secure Boot state (VERIFIED in `src/boot/stub.c`). The confidential-VM
   gate exists because the SMBIOS tables are supplied by the host/hypervisor and
   are NOT covered by the guest's attestation -- the stub comment is explicit:
   "SMBIOS OEM Strings data is controlled by the host admin and not covered by the
   VM attestation, so MUST NOT be trusted when in a confidential VM". So the stub
   refuses the string exactly where the host sits OUTSIDE the trust boundary (a
   confidential VM: AMD SEV/-ES/-SNP, Intel TDX -- detected in `src/boot/vmm.c` via
   CPUID + the SEV MSR); in an ordinary VM the host already owns guest memory (it is
   IN the TCB), so a host-supplied cmdline grants no new power and is honored. Its
   only protection then is measurement into PCR 12 -- an untrusted hypervisor can
   inject arguments. Treat it as a trusted-host / debug convenience, NOT a
   Secure-Boot-safe mechanism. (It is the one option here that does not preserve the
   SB authentication guarantee.)

So Kicksecure's answer for users: ship signed addons for supported toggles (or
re-sign the UKI via the kernel hook); never rely on SMBIOS for anything
security-relevant. These are upstream systemd mechanisms -- Debian ships
`ukify`/`kernel-install` but documents no addon wrapper, so the addon-signing
tooling (and an own trusted key) is Kicksecure's to provide. (Credential addons
`<uki>.efi.extra.d/*.cred` exist too, but carry data, not cmdline.)

GRUB's `smbios` command on its own only sets a GRUB variable -- it does not reach
the kernel cmdline unless a distro interpolates that variable into
`GRUB_CMDLINE_LINUX`. Kicksecure's derivative-maker does exactly that, as an
OPT-IN, dangerous-by-design CI/boot-test feature (the `--smbios-reader` build flag;
default OFF): an `/etc/grub.d` snippet reads SMBIOS Type 1 (only when the
manufacturer is `QEMU`), extracts a `dm-cmdline=...` prefix from the serial-number
string via `regexp` into `${dm_smbios_extra}`, and appends that to the kernel
cmdline (also enabling a serial console and instant boot). It is the GRUB analogue
of systemd-stub's `io.systemd.stub.kernel-cmdline-extra`. So GRUB CAN ingest a
host-supplied SMBIOS string as kernel arguments -- but only because the DISTRO
wires it, and Kicksecure treats that wiring as a backdoor: off by default, gated
behind a dangerous-option confirmation, deleted from any image not built with the
flag, and documented "quite literally backdoored by design ... [to] NEVER be
created or used except in very specific testing scenarios."

That reader does NOT honor a confidential-VM check, and (unlike systemd-stub) it
does not need one: systemd ships its SMBIOS reader ENABLED in every UKI, so it must
refuse the input at runtime wherever the host is untrusted (the
`is_confidential_vm()` guard); Kicksecure's reader is never present in a production
or confidential-VM image in the first place -- the control is build-time EXCLUSION,
not a runtime gate. A runtime CVM gate would be hard to add anyway: a Secure-Boot
grub.cfg has no robust confidential-VM primitive. GRUB's `cpuid` command exposes
only long-mode/PAE, not the SEV leaf `0x8000001f` or the TDX leaf `0x21`; there is
no `is_confidential_vm` equivalent; and the MSR route (`rdmsr 0xC0010131`) depends
on the signed `grubx64.efi` actually shipping the `rdmsr` module, which distros
normally curate OUT of the signed core (the real Secure-Boot barrier -- NOT GRUB
lockdown, which blocks only the write/leak primitives `wrmsr`/`iorw`-write/`memrw`,
not `rdmsr`). TDX is thus undetectable from grub.cfg at all; SEV only non-robustly
and essentially BIOS-only. (The `QEMU`-manufacturer gate is also not a security
boundary -- the host sets that SMBIOS string.)

GRUB's broader, always-present gap is the one this note returns to throughout: its
kernel cmdline comes from an UNSIGNED `grub.cfg`/`grubenv` (and the interactive `e`
editor) on host-controlled storage, authenticated only by `check_signatures=enforce`
or by baking the config into the signed `core.img`, and otherwise merely MEASURED
into PCR 8 (the `tpm` module logs each `kernel_cmdline:` string). That
unsigned-config family -- CVE-2020-10713 "BootHole" (Red Hat: grub.cfg parsing
bypasses Secure Boot), Debian bug #906124 (the `check_signatures`/`pgp` regression
and its own SB-bypass hazard) -- is exactly what the UKI's single signed-and-measured
image is meant to close. So the GRUB-world answer to "change the cmdline safely" is
the own-GRUB design (own key in `core.img` + `check_signatures` over a signed
`grub.cfg`), not a signed-addon or SMBIOS mechanism.


Sources: systemd-stub (addons, SMBIOS, cmdline override) https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html ; ukify (--cmdline, addon mode, signing) https://www.freedesktop.org/software/systemd/man/latest/ukify.html ; kernel-install cmdline sources https://www.freedesktop.org/software/systemd/man/latest/kernel-install.html ; loader.conf editor https://www.freedesktop.org/software/systemd/man/latest/loader.conf.html ; UKI spec (addons) https://uapi-group.org/specifications/specs/unified_kernel_image/ ; systemd stub.c (SMBIOS/cmdline gating, verified) https://github.com/systemd/systemd/blob/main/src/boot/stub.c ; systemd vmm.c (is_confidential_vm, CPUID/SEV-MSR detection) https://github.com/systemd/systemd/blob/main/src/boot/vmm.c ; commit adding the confidential-VM SMBIOS guard https://github.com/systemd/systemd/commit/4b1153cfcc397df5a095f2ec7e587787e8ba47ee ; Kicksecure derivative-maker opt-in SMBIOS cmdline reader (--smbios-reader): packages/kicksecure/vm-config-dist/etc/grub.d/01_smbios-reader, etc/default/grub.d/99_smbios-cmdline.cfg, iso-build-data/grub-config/smbios-reader.cfg (repo Kicksecure/derivative-maker) ; GRUB cpuid.c (long-mode/PAE only) https://git.savannah.gnu.org/cgit/grub.git/tree/grub-core/commands/i386/cpuid.c ; GRUB rdmsr.c (not lockdown-gated) https://git.savannah.gnu.org/cgit/grub.git/tree/grub-core/commands/i386/rdmsr.c ; GRUB lockdown framework https://www.gnu.org/software/grub/manual/grub-dev/html_node/Lockdown-framework.html ; CVE-2020-10713 BootHole (grub.cfg unauthenticated, Red Hat) https://access.redhat.com/security/vulnerabilities/grub2bootloader ; Debian bug #906124 (grub check_signatures/pgp) https://bugs.debian.org/906124

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
  DELETES the platform key `PK`, wipes KEK/db, and pushes the Microsoft
  certificates into `dbx` (the deny-list -- it OVERRIDES `db`, so adding MS there
  forbids MS-signed binaries DURABLY, unlike merely dropping MS from `db`, which a
  later firmware/vendor update or KEK-authorized `db` change could silently
  re-add), enrolls only the owner's keys into db, and re-takes ownership with a
  fresh ephemeral `PK`. The owner's key becomes the sole trust root. Because the
  MS-signed shim no longer verifies under sovereign keys, this also means DROPPING
  shim -- which REMOVES the MOK layer entirely (MOK, MokManager, `MokListRT` and
  `SHIM_LOCK` are shim features). Trust then roots in your own keys in UEFI `db`,
  loaded by the kernel directly into `.platform`, and dm-verity roothash
  signatures anchor via `CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG_PLATFORM_KEYRING`,
  not the MOK/`.machine` path the default uses.

These are NOT one spectrum of the same mechanism: the default roots trust in
Microsoft's CA (via the MS-signed shim) with the owner's key ADDED through MOK,
whereas Sovereign Boot roots trust in the owner's own `db` keys and REPLACES the
MOK layer rather than extending it. The honest shape: ship MS-shim-compatible by
default (works everywhere; the owner may add their key via MOK to sign their own
UKIs), and present Sovereign Boot as a deliberate migration to a different trust
root. Its costs: the whole chain must be signed by the owner's key; dual-boot /
vendor recovery / firmware-capsule paths relying on MS signatures break; and the
shim question is awkward -- Debian's only shim is Microsoft-CA-signed, so under
sovereign keys you must either trust it BY HASH in `db` (fragile: re-enroll on
every `shim-signed` update) or RE-SIGN it with your own key; the cleanest design
simply DROPS shim and boots an own-signed loader straight from `db`. (Debian bug
#1102680 is an open request for a Debian-CA-signed shim that would ease a
Debian-key sovereign path.) The kernel holds NO built-in Microsoft key -- MS is
trusted only because it sits in UEFI `db` -- so removing MS from `db` (and adding
it to `dbx`) drops MS trust chain-wide, kernel included. Sovereign Boot also
sidesteps the mid-2026 Microsoft 2011-CA expiry (the replacement UEFI CA 2023
must reach each machine's db via firmware/OS updates; an owner rooting trust only
in their own db does not depend on that rotation). Kicksecure's own Keys section
has not committed to a MOK design; its forward model is user-controlled key
enrollment confirmed in firmware (TOFU + fingerprint display + multi-key
rotation) -- the Sovereign-Boot end of this. Status: Sovereign Boot is
RC/demo-stage (3mdeb spec v0.1.0), not a shipped default.

**Kicksecure requirement: keep MOK functional -- and what it costs.** MOK is a
shim feature (MokManager, `MokListRT`, `SHIM_LOCK`), so keeping it means keeping
shim -- the "drop shim" full-sovereign path is ruled out. The catch, source-proven:
Debian's shim has the **Debian Secure Boot CA compiled in as its `VENDOR_CERT`**
(`debian/rules` sets `VENDOR_CERT_FILE=debian/debian-uefi-ca.der`; shim's `cert.S`
`.incbin`s it into the `.vendor_cert` PE section; `verify.c check_allowlist()`
runs `AuthenticodeVerify(..., vendor_cert, ...)` as a standalone accept path), and
shim verifies the next stage against that embedded CA INDEPENDENT of `db`/MOK. So
ANY shim derived from Debian's -- re-signed, dual-signed, or a Debian-CA-signed
shim (#1102680) -- still trusts stock Debian-signed GRUB, and an attacker drops
the publicly-available Debian-signed GRUB + an unsigned `grub.cfg` + an evil
initrd and boots it. **Keeping Debian's shim therefore gives DETECTION (and
removes the MS dependency for who launches shim), but NOT loader-level
PREVENTION.** The options:

- **(a) Default -- Debian's MS-signed shim + additive MOK.** MOK works; the owner
  signs their own UKIs/modules. Detection-level; Microsoft AND Debian both remain
  trust roots.
- **(b) Dual-signed Debian shim** -- append the owner's signature as an extra
  `WIN_CERTIFICATE` to Debian's shim. Authenticode's digest excludes the cert
  table, so the MS signature survives and no MS/Debian secret is needed; append
  with `systemd-sbsign`/`pesign` (which ADD a cert-table entry), NOT classic
  `sbsign` (replaces) nor `osslsigncode -nest` (the Windows nested form UEFI does
  not read). This lets the SAME shim boot on MS-key machines AND on machines where
  the owner key is ADDED to `db` -- a compatibility convenience. But it gives
  neither sovereignty nor prevention: it still carries the Debian CA (same
  evil-initrd hole as (a)), and it does NOT boot on a FULL-sovereign machine that
  pushed the MS CA into `dbx`, because EDK2 rejects the whole image when ANY
  signature chains to a `dbx` cert (`DxeImageVerificationLib` `IsForbiddenByDbx`
  -> `EFI_ACCESS_DENIED`). Microsoft (2011+2023), Fedora and openSUSE ship
  multi-signed shims, but those are MS+MS or MS+distro signatures, not
  MS+sovereign-owner; the owner-plus-MS case is an extrapolation and breaks
  exactly at the MS-in-`dbx` end.
- **(c) Own shim -- the prevention-WITH-MOK path (the goal).** Build a Kicksecure
  shim whose `VENDOR_CERT` is the KICKSECURE key, so it chainloads ONLY
  Kicksecure-signed stages -- stock Debian-signed GRUB is rejected, cutting the
  evil-initrd path, while the shim still provides the full MOK layer (MOK is
  shim-inherent, independent of which cert is baked in). The next stage is a
  Kicksecure-signed UKI (`ukify` bundles Debian's STOCK vmlinuz+initrd+cmdline,
  signed once; the initrd is inside the signed envelope) over a dm-verity root --
  less effort and more robust than own-GRUB+`check_signatures`. Two complementary
  delivery modes: (i) MS-signed own shim via rhboot/shim-review (boots out-of-box
  on stock Microsoft-key machines; a small Debian derivative, [Lernstick](https://www.bfh.ch/en/research/research-areas/lernstick/),
  is precedent -- it ships its OWN MS-signed shim built on Debian's GRUB and
  kernel (shim-review [issue #528](https://github.com/rhboot/shim-review/issues/528),
  2026) -- and Kicksecure's prevention goal answers shim-review's "why not reuse
  Debian's shim"); (ii) Sovereign own-key -- the owner enrolls Kicksecure's
  PK/KEK/db via the Dasharo/3mdeb wizard and the SAME Kicksecure-signed shim/UKI
  boots natively, no Microsoft. Ongoing cost (the real one): owning a shim means
  owning its SBAT generation and the shim/GRUB-CVE treadmill -- rebuild on the new
  upstream shim, bump SBAT, re-run shim-review, re-sign, and ship the fixed shim
  BEFORE the revocation lands -- plus HSM key custody and a monitored security
  contact.
- **(d) Full Sovereign Boot** -- drop shim, own `db` keys, MS (and, for real
  prevention, Debian's CA too) in `dbx`, sign only own UKIs. Prevention, but MOK
  is gone, so out-of-tree module signing would then need the kernel-rebuild route
  -- off the table per "keep MOK".

Honest bottom line: keep-MOK (keep shim) and loader-PREVENTION reconcile only by
owning the shim (c); (a)/(b) are detection-level. Near-term for Kicksecure that
means DETECTION -- measured boot + TPM-sealed FDE (with a recovery key) +
attestation -- optionally with (b) for compatibility; true prevention is the
own-shim (c) project.

Dual-signing operational cost (if (b) is used): the appended signature lives in
that specific shim binary, so a Debian `shim-signed` update ships a FRESH binary
without it -- a dpkg trigger / apt hook must re-append the owner signature on
every update (mechanical, no MS secret, same pattern as DKMS MOK signing).

Dual-signing caveats: a MINORITY of firmware does not iterate all cert-table
signatures and rejects multi-signed images (documented on some ASUS boards) --
there the second signature must be stripped. And `dbx`-by-hash is fatal
regardless: if that shim build is ever hash-revoked (the standard post-CVE
lever), no signature helps -- which also sinks the "hash-pin the shim" idea. The
Kicksecure wiki's old "an EFI binary cannot have two signatures" line is already
marked outdated; multi-signature is standards-blessed (UEFI authenticates an
image if ANY cert-table signature chains to `db` AND nothing is in `dbx`) and in
production.

Can you dodge the `dbx`-by-hash problem by just not applying revocations? Mostly
NO, and the part that is yes you get for free. The real control is NOT a package
but the KEK: a `dbx` write is an authenticated variable update that firmware
accepts only if KEK-signed, so once a sovereign owner replaces or wipes the
Microsoft KEK, Microsoft's `dbx` updates cannot be authorized at all -- MS
revokes only MS-ecosystem binaries a sovereign machine never boots anyway, so
those entries are genuinely moot (the "obsolete anyhow" intuition, correct for
exactly this case). But (a) on Debian there is no dbx "package" to decline:
revocations are applied by fwupd/LVFS (`org.uefi.dbx`), and application is already
opt-in (`fwupd-refresh` only fetches metadata); `secureboot-db` is Ubuntu-only.
(b) Any `dbx` entry ALREADY applied persists in firmware NVRAM (append-only, no
downgrade) -- declining future updates does not remove it. (c) Deliberately
skipping the revocation of YOUR OWN shim build to keep it bootable means running
a bootloader with a known Secure Boot bypass -- a footgun; the sound fix is
roll-forward to a fixed shim and re-dual-sign (the re-sign trigger), not freezing
on a revoked build.

Why MOK is the deciding factor (so shim is NOT optional): the case that forces it
is user / out-of-tree KERNEL MODULE signing -- DKMS modules (VirtualBox, tcp-isn,
CPU-info-leak, ...) signed today via `/var/lib/dkms/mok.key`. On a stock Debian
kernel under Secure Boot, an owner key enrolled in `db` reaches only the
`.platform` keyring, which the kernel DELIBERATELY excludes from module-signature
verification -- it checks `.builtin` and `.secondary`, into which only
`.machine`/MOK is linked (`db` holds MS/vendor keys the kernel must not let sign
modules). Note the asymmetry: `.platform`/`db` DOES anchor dm-verity roothash and
kexec, but NOT modules. So without MOK an out-of-tree `.ko` cannot be trusted
except by rebuilding the kernel with the key in `CONFIG_SYSTEM_TRUSTED_KEYS`,
which abandons riding Debian's stock kernel. And Kicksecure does NOT plan to
build its own kernel at this time, so that rebuild escape hatch is explicitly off
the table -- leaving MOK (hence shim) as the ONLY path for user module signing.
Hence the conditional "if sovereign boot can replace MOK, shim is optional"
resolves NEGATIVE for Kicksecure: on a stock kernel it cannot, so shim stays.
(Module-sig enforcement only bites under
Secure Boot -- `CONFIG_MODULE_SIG_FORCE` is unset, lockdown enforces -- and the
MOK path needs `CONFIG_INTEGRITY_MACHINE_KEYRING=y`, present in trixie. An
upstream RFC `CONFIG_MODULE_SIG_PLATFORM` (2025, not merged) would let `db` sign
modules and would make shim droppable -- worth tracking.)


Sources: shim README https://github.com/rhboot/shim ; rodsbooks controlling Secure Boot https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html ; kernel module signing https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html ; Ubuntu Secure Boot https://documentation.ubuntu.com/security/docs/security-features/platform-protections/secure-boot/ ; Kicksecure Sovereign Boot https://www.kicksecure.com/wiki/Sovereign_Boot ; Dasharo wizard https://docs.dasharo.com/projects/sovereign-boot-wizard/ ; 3mdeb verified-boot app spec https://github.com/3mdeb/verified-boot/blob/master/verified_boot_app_spec.md ; MS 2011 CA expiry / UEFI CA 2023 https://eclypsium.com/blog/microsoft-secure-boot-certificates-expire-2026/ ; Debian bug #1102680 https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1102680 ; Kicksecure Verified_Boot Keys https://www.kicksecure.com/wiki/Verified_Boot#Keys ; Authenticode digest excludes cert table https://blog.trailofbits.com/2020/05/27/verifying-windows-binaries-without-windows/ ; Fedora multi-signed shim https://fedoraproject.org/wiki/Test_Day:2026-01-12_Multi-signed_shim ; openSUSE dual-signed shim + ASUS https://forums.opensuse.org/t/secure-boot-asking-for-certificate-permission/132905 ; systemd-sbsign appends https://manpages.opensuse.org/Tumbleweed/systemd-experimental/systemd-sbsign.1.en.html ; pesign https://man.archlinux.org/man/pesign.1 ; .machine keyring patch https://kernsec.org/pipermail/linux-security-module-archive/2022-January/030363.html ; DKMS MOK workflow https://www.kicksecure.com/wiki/Secure_Boot ; CONFIG_MODULE_SIG_PLATFORM RFC https://lkml.iu.edu/hypermail/linux/kernel/2506.0/01180.html ; Debian shim packaging VENDOR_CERT https://salsa.debian.org/efi-team/shim/-/raw/master/debian/rules ; shim verify.c https://raw.githubusercontent.com/rhboot/shim/main/verify.c ; shim cert.S https://raw.githubusercontent.com/rhboot/shim/main/cert.S ; EDK2 DxeImageVerificationLib https://github.com/tianocore/edk2/blob/master/SecurityPkg/Library/DxeImageVerificationLib/DxeImageVerificationLib.c ; Debian trust-machine-keyring-by-default patch https://sources.debian.org/src/linux/6.12.107-1/debian/patches/features/all/db-mok-keyring/trust-machine-keyring-by-default.patch

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
validated on stock trixie. The tests do NOT establish CONFINEMENT -- and per the
design section no loader provides it while Debian/MS keys are trusted -- so the
real guarantee is measured boot + TPM-sealing/attestation (detection), with
Sovereign Boot for loader-level prevention.


Sources: Debian grub signed-image modules https://salsa.debian.org/grub-team/grub/-/raw/master/debian/build-efi-images ; Debian SecureBoot https://wiki.debian.org/SecureBoot ; IPE (Integrity Policy Enforcement) LSM https://docs.kernel.org/admin-guide/LSM/ipe.html

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
   then REPLAYS the logs: it folds each event's RECORDED digest into
   `PCR = H(PCR || digest)` in order (skipping non-extended `EV_NO_ACTION`
   records such as the Spec ID event) and checks the result equals the quoted
   PCRs -- proving the log is the unmodified one that produced the quote. It then
   validates the event CONTENT against a reference policy; content can be
   recomputed-and-matched to the digest only for event types whose digest IS a
   hash of that content (GRUB's PCR 8/9 string/file measurements, UEFI variable
   events), NOT for PE image-load events (`EV_EFI_BOOT_SERVICES_APPLICATION`)
   whose digest is the Authenticode hash of the binary, not of the event body.
   (A naive "recompute every digest from the event body" would reject a genuine
   boot on those events.)

Keylime works this way (quote over PCRs + `binary_bios_measurements` + the IMA
log; it replays by folding the recorded digests, then applies a measured-boot
policy and a runtime IMA policy to the log content). GRUB on EFI
extends PCR 8/9 through the firmware's `EFI_TCG2_PROTOCOL HashLogExtendEvent`,
so a verifier sees firmware + Secure Boot + GRUB in one continuous log.

The systemd-boot + UKI path gives the cleaner PCR 11 as well (better
reference-state stability than PCR 8/9). Either way the attestation MODEL is
sound; the roughness in the GRUB PCR 8/9 path is operational (brittle reference
state), not theoretical.

Sources: TPM2 PCR measurements https://systemd.io/TPM2_PCR_MEASUREMENTS/ ; Linux TPM PCR registry https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/ ; Keylime measured boot https://keylime.readthedocs.io/en/latest/user_guide/use_measured_boot.html ; Keylime https://keylime.dev

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


Sources: Ubuntu confidential VMs on Azure https://ubuntu.com/blog/ephemeral-ubuntu-confidential-vms-azure ; Azure guest attestation https://learn.microsoft.com/azure/confidential-computing/guest-attestation-confidential-vms ; SUSE SLE Micro attestation https://documentation.suse.com/sle-micro/5.5/html/SLE-Micro-all/cha-security-attestation.html ; Keylime install (RHEL/SUSE only) https://keylime.readthedocs.io/en/latest/installation.html

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


Sources: IMA concepts https://ima-doc.readthedocs.io/en/latest/ima-concepts.html ; IMA policy https://ima-doc.readthedocs.io/en/latest/ima-policy.html ; debhelper bug #766267 https://bugs.debian.org/766267 ; apt bug #823945 https://bugs.debian.org/823945 ; apt-devel patch thread https://lists.debian.org/deity/2016/05/msg00034.html ; DebSignatures spec https://wiki.debian.org/Teams/Dpkg/Spec/DebSignatures ; Fedora rpm-plugin-ima https://packages.fedoraproject.org/pkgs/rpm/rpm-plugin-ima/ ; ima-evm-utils https://packages.debian.org/sid/ima-evm-utils

## Installed system: verity root + working apt

The design must cover the installed system, not only the live ISO. The tension:
a dm-verity root is read-only and immutable (a single signed root hash), but apt
wants to write `/usr`, `/etc`, `/var`, and the dpkg database. Every shipping
image-based system resolves this by moving the write OUT of the running system.
Three patterns:

- (a) **A/B image swap** (Vanilla OS/ABRoot, Ubuntu Core, bootc): never `apt
  install` into the live root; build/pull a new signed image and reboot.
  Strongest atomicity, weakest "it is still apt" feel.
- (b) **Immutable /usr + writable /etc,/var overlay** (systemd verity /usr via
  `usrhash=` / `mount.usr=dissect`, sysext/confext): the vendor tree is sealed,
  machine state is writable. Clean integrity story, but no on-device package
  manager (sysext is not apt: additive, no dependency resolution).
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


Sources: transactional-update https://github.com/openSUSE/transactional-update

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

The goal is PREVENTION (Sovereign / Secure Boot level), not merely detection. The
only loader-level prevention that KEEPS MOK is an OWN Kicksecure shim (option (c)
in the key-ownership section): a shim whose `VENDOR_CERT` is the Kicksecure key,
chainloading a Kicksecure-signed UKI (Debian's stock kernel+initrd bundled by
`ukify`, signed once) over a dm-verity root. Because the shim trusts only the
Kicksecure cert, stock Debian-signed GRUB is rejected -- the evil-initrd path is
cut -- and MOK keeps working for DKMS/out-of-tree modules (MOK is shim-inherent).

Deliver it both ways, from the same Kicksecure-signed artifacts:
- **MS-signed own shim via shim-review** -- prevention that ALSO boots out-of-box
  on stock Microsoft-key machines. Precedent exists for small Debian derivatives
  ([Lernstick](https://www.bfh.ch/en/research/research-areas/lernstick/), whose
  own shim is built on Debian GRUB + a trixie kernel). Cost: an upstream-based
  reproducible build, HSM key custody, a
  verified security contact, and the recurring SBAT/CVE re-review-and-re-sign
  treadmill.
- **Sovereign own-key (Dasharo/3mdeb wizard)** -- the owner enrolls Kicksecure's
  PK/KEK/db (MS, and for full sovereignty Debian's CA, into `dbx`); the same
  Kicksecure-signed shim/UKI then boots natively, no Microsoft -- maximal owner
  control, at the cost of out-of-box compatibility.

Layer DETECTION on top as defense-in-depth, NOT as the goal: measured boot +
TPM-sealed FDE. Seal the FDE key to PCR 7 (+11), not the volatile 8/9, via a
signed `pcrlock` prediction, and keep an offline recovery key. (A surviving
passphrase keyslot stays capturable, so sealing withholds the auto-unlock secret
and aids attestation but does not itself prevent -- prevention comes from the
own-shim chain above.)

Do NOT rely on loader choice (GRUB vs systemd-boot) or on any Debian-DERIVED shim
for confinement: Debian's shim embeds the Debian CA, so re-signing/dual-signing
it still trusts Debian-signed GRUB (evil-initrd path open), and an own-signed
GRUB + `check_signatures` is likewise bypassed by swapping in Debian's GRUB.
Prevention requires the OWN shim (own `VENDOR_CERT`), full stop.

Start IMA at measurement-only (`ima_policy=tcb` + Keylime); defer appraisal+EVM,
or a local-hash dpkg hook, to a frozen-appliance scope until Debian gains
package-level file signing.

Sources: rhboot/shim-review https://github.com/rhboot/shim-review ; Lernstick (Debian-based live system, BFH) https://www.bfh.ch/en/research/research-areas/lernstick/ and its shim-review precedent https://github.com/rhboot/shim-review/issues/528 ; Dasharo/3mdeb Sovereign Boot wizard https://docs.dasharo.com/projects/sovereign-boot-wizard/ ; shim SBAT lifecycle https://sources.debian.org/src/shim/16.1-1/SBAT.md ; MS UEFI CA 2023 (post-2026-06-27 signing) https://lwn.net/Articles/1079808/


## Open product decisions

- **How to deliver prevention (the goal), not whether.** Prevention needs an OWN
  Kicksecure shim (own `VENDOR_CERT`) + Kicksecure-signed UKI over dm-verity,
  keeping MOK. The open choice is the delivery mix: MS-signed own shim via
  shim-review (out-of-box, but the SBAT/CVE re-sign treadmill) and/or Sovereign
  own-key enrollment (no MS, owner takes ownership). Recommended: pursue both --
  shim-review for stock machines, the Dasharo/3mdeb wizard for sovereign owners --
  from one set of Kicksecure-signed artifacts, with measured boot + TPM-sealed FDE
  as defense-in-depth. (Three earlier drafts -- "stock GRUB -> UKI", "systemd-boot
  confines boot", and "re-signed/dual-signed Debian shim prevents" -- were all
  withdrawn: no Debian-derived shim prevents.)
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
