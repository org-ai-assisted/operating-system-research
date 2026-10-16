# Persistent storage on a booted live USB

How do you give a live USB persistent storage -- ideally whole-root
persistence with full-disk encryption -- on the SAME medium the system
booted from? This note surveys how the major projects solve it, identifies
the one constraint that rules out the naive approach, and records the design
chosen for Kicksecure/Whonix.

Scope: Kicksecure/Whonix use dracut `dmsquash-live` with an overlayfs union
root. The findings generalize, but the decision at the end is specific to
that stack.


## The core constraint: whole-disk iso9660 blocks in-place partitioning

The obvious idea -- flash the distributed isohybrid ISO with `dd`/Etcher,
boot it, then carve a persistence partition into the USB's trailing free
space and `mkfs`/`cryptsetup` it -- does not work, and the reason is
structural:

- An isohybrid ISO written raw to a whole disk puts an `iso9660` filesystem
  on the WHOLE block device (`/dev/sdX`), not on a partition. The filesystem
  label lands on the whole disk, so the live medium is mounted from
  `/dev/sdX` directly.
- With the whole disk mounted, the kernel refuses `O_EXCL` opens of any
  partition carved out of that disk: `mkfs.ext4` reports "in use by the
  system" (even `-F` is refused), and `cryptsetup luksFormat` refuses with no
  override. `wipefs --force` works, but formatting and LUKS do not.
- dracut's own boot-time persistence-carving module
  (`70dmsquash-live-autooverlay`) confirms the constraint: it EXPLICITLY skips
  read-only and whole-disk-iso9660 media (`ro != 0` or unpartitioned ->
  "Skipping overlay creation"). It only carves free space on a real,
  writable, partitioned device.

A confirmed-working but non-idiomatic workaround exists (see "Loop-device
indirection"), but every project that actually ships persistence avoids the
problem instead of working around it: none of them raw-dd a whole-disk
isohybrid when persistence is wanted.


## How the major projects solve it

| Project | Medium layout | Persistence unit | Encryption | Provisioned |
|---|---|---|---|---|
| Tails | partitioned GPT (files copied, not dd); FAT32 system partition + reserved free space | dedicated LUKS2 partition (`TailsData`, ext4 inside) | mandatory LUKS2 + Argon2id | at runtime, by a wizard inside the booted system |
| Debian live-boot | user's choice: file on a writable fs, or a labeled partition | overlay FILE (`persistence` + `persistence.conf`) or partition | native `persistence-encryption=luks` | at prep time; discovered at boot by label/file |
| Fedora `livecd-iso-to-disk` | installer partitions + copies files (ext4/vfat/...), not dd | overlay FILE `/LiveOS/overlay-*` (dm-snapshot) or `--overlayfs` dirs | `/LiveOS/home.img` LUKS by default; root overlay unencrypted | at install time |
| Ubuntu casper | SDC/mkusb copy files to a FAT partition (+ a partition for the writable area) | `casper-rw`/`writable` FILE (<=4 GB) or labeled partition | none native (DIY LUKS) | at install time; discovered at boot by label |
| dracut `dmsquash-live` | whole-disk iso9660 for a raw-dd ISO; a partition/file on writable media otherwise | overlay FILE (`rd.overlay=LABEL=x:/path`, `:auto` -> `/LiveOS/overlay-<label>-<uuid>`) or directory | via `rd.luks.*` on the backing device | overlay file auto-created at boot only on writable media |

The common thread: three of the four shipping projects deliberately do NOT
put an iso9660 on the whole disk. They either (a) keep persistence as a FILE
on an already-writable filesystem, or (b) create the medium as a PARTITIONED
device via an installer (file-copy, not raw dd) so the live system lives on a
partition and the whole disk is never mounted.


### Tails -- LUKS partition, runtime wizard

The Tails installer/cloner writes a partitioned GPT device: a FAT32 system
partition labeled `TAILS` (with the GPT hidden flag so Windows ignores it),
files copied from the ISO (NOT a raw dd), and the rest of the device left
unpartitioned. The Persistent Storage wizard, run from inside the booted
system, carves a dedicated LUKS2/Argon2id partition (`TailsData`, ext4
inside) into that reserved free space. Runtime partitioning succeeds
precisely because the system is on a FAT32 partition, so the whole disk is
never mounted.

- https://tails.net/doc/persistent_storage/
- https://tails.net/contribute/design/persistence/
- https://tails.net/contribute/design/installation/


### Debian live-boot -- file or partition, by label

live-boot accepts either an overlay image FILE on any readable filesystem, or
a partition with the GPT name / filesystem label `persistence`. In both cases
the volume's root must contain a `persistence.conf` listing what to persist
(`/home`, `/var/cache/apt`, or `/ union` for the whole root), else the volume
is ignored. Boot params: `persistence`, `persistence-label=<label>`,
`persistence-encryption=luks`.

- https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-run-time-behaviours.en.html
- https://manpages.debian.org/buster/live-boot-doc/persistence.conf.5


### Fedora -- overlay FILE via livecd-iso-to-disk

`livecd-iso-to-disk` is an installer, not a raw copy: it creates a partition
table, formats a writable filesystem (ext4 default; also vfat/btrfs/xfs/f2fs),
and copies the ISO's `LiveOS/` contents onto it. Persistence is a FILE,
`/LiveOS/overlay-<label>-<uuid>`, sized with `--overlay-size-mb`:

- Classic dm-snapshot CoW file: write-once difference tracking -- every write
  AND delete consumes space irreversibly until the file is exhausted, then the
  overlay flips read-only (`dmsetup status live-rw` shows headroom). A known
  footgun.
- `--overlayfs`: OverlayFS upper/work directories instead of a CoW file; on a
  non-vfat fs it can grow to device capacity with no write-once exhaustion.

`/home` can get its own `/LiveOS/home.img` (`--home-size-mb`),
LUKS-encrypted by default; the root overlay itself is unencrypted by default.
Fedora Media Writer, which does a raw dd / isohybrid, explicitly does NOT
support persistence and tells users to use `livecd-iso-to-disk` instead.

- https://docs.fedoraproject.org/en-US/quick-docs/creating-and-using-a-live-installation-image/
- https://www.mankier.com/8/livecd-iso-to-disk


### Ubuntu -- casper-rw file or partition

casper looks for `casper-rw`/`writable`/`home-rw` as either a loop FILE on a
writable filesystem or a labeled partition; the first becomes the writable
union branch (`/cow`). A file is capped at 4 GB (FAT limit); a partition
removes the cap. Startup Disk Creator / mkusb copy files to a FAT partition
and add the writable area; they do not raw-dd. No first-class native LUKS
persistence -- encryption is DIY. Since casper 1.439 the preferred label is
`writable` (an ext4 partition).

- https://manpages.ubuntu.com/manpages/focal/man7/casper.7.html
- https://launchpad.net/ubuntu/+source/casper/1.439


### dracut dmsquash-live -- the mechanics underneath

`rd.overlay=` (the current spelling; `rd.live.overlay=` is the deprecated
alias) takes `<devspec>[:{<pathspec>|auto}]`. The overlay target can be:

- a FILE on a labeled filesystem, loop-mounted and used as a dm-snapshot CoW
  image or (if it holds `overlayfs/`+`ovlwork/` dirs) an OverlayFS backing;
- a DIRECTORY with a sister `../ovlwork`, symlinked into `/run/overlayfs` for
  OverlayFS directly;
- absent / empty / `auto` -> defaults to `/<rd.live.dir>/overlay-<label>-<uuid>`
  on the live medium (`rd.live.dir` defaults to `LiveOS`).

So the idiomatic dracut persistent overlay is a FILE (or directory) on a
WRITABLE medium. For a raw-dd isohybrid the medium is read-only iso9660 on the
whole disk, so no overlay file can be written and no partition can be carved
in place -- the exact constraint above. When `rd.overlay=` is absent or the
overlay is missing/unwritable, dracut falls back to a temporary overlay in the
initramfs tmpfs (RAM), lost on shutdown.

Source: dracut-ng `modules.d/70dmsquash-live/dmsquash-live-root.sh`,
`parse-dmsquash-live.sh`, `modules.d/70overlayfs/overlayfs-lib.sh`,
`modules.d/70dmsquash-live-autooverlay/create-overlay.sh`,
`man/dracut.cmdline.7.adoc`.


## Approaches considered (and verdicts)

1. Runtime partition carve on the raw-isohybrid USB -- BLOCKED by the
   whole-disk iso9660 O_EXCL constraint above. The naive target.

2. Loop-device indirection over the carved partition's byte region --
   `losetup --offset/--sizelimit` a loop device over the new partition's byte
   range on the mounted whole disk, then `mkfs`/`cryptsetup` the loop.
   CONFIRMED working empirically. But non-idiomatic, fragile (TOCTOU window,
   partition-offset math from `/sys/class/block/<part>/start`), and it is the
   exact medium dracut's own autooverlay refuses. Fallback only.

3. Tails model -- partitioned GPT medium + runtime LUKS partition carve.
   Idiomatic for amnesic-by-default with opt-in encrypted persistence. The
   whole disk is never mounted, so the carve works with no workaround. Matches
   a "whole root persists in trailing free space, same label for a second
   disk" requirement.

4. Fedora model -- overlay FILE on a writable medium. Idiomatic for a
   convenience persistence feature. No repartitioning. But the classic
   dm-snapshot file has an overflow-to-read-only failure mode, the root
   overlay is unencrypted by default, and a file maps poorly to "trailing free
   space" and "a second whole disk via the same label".

5. dracut autooverlay -- boot-time new-partition carve, but gated to writable
   partitioned non-iso9660 media; confirms the constraint rather than solving
   it for raw-dd.

6. Appliance model -- ship a pre-installed normal disk image. See below. This
   is the chosen direction.


## Decision for Kicksecure/Whonix: the appliance model

Requirements that shaped the choice: whole-root persistence that behaves like
a normal install; full-disk encryption (not just `/home`); encryption the
default; a second disk as an option; and -- a late reframing -- amnesic boot
is NOT required as the default, because `grub-live` already provides
amnesic-on-demand from the GRUB menu on any installed system.

Given that, the live-overlay models (3, 4) are the wrong frame. The chosen
design ships a PRE-INSTALLED normal Kicksecure disk image (`.img`, the same
kind of partitioned install dracut/Debian produce for VM images), flashed to
the USB. It boots as a genuinely normal persistent install; `grub-live`
supplies amnesic mode when wanted.

Why this is better for these requirements:

- It is a real install, so "fully installed, normal, persistent" is literally
  true -- no overlayfs whiteouts, no double-stored upgrades, no union
  accounting.
- Full-disk encryption is standard LUKS on the root, which only a normal root
  (not an overlay) supports. `cryptsetup reencrypt --encrypt` is the idiomatic
  way to add FDE to an existing install.
- Kernel upgrades are normal: the new kernel lands in `/boot` and GRUB reads
  it. No bespoke squashfs-kernel sync hook is needed.
- The whole-disk iso9660 O_EXCL problem, the loop-indirection workaround, and
  a `search --label -> configfile` boot dispatch all disappear -- a normal GPT
  disk is never whole-disk-iso-mounted.
- Secure Boot is the standard Debian layout: unencrypted `/boot` (GRUB cannot
  do luks2/cryptodisk in the shipped signed build), LUKS root unlocked by the
  initramfs. Boring and signed.

### Layout

- ESP + plaintext `/boot` + LUKS2/Argon2id root.
- Generic initramfs (`hostonly=no`) so the USB boots on any machine.
- `systemd-repart` with `GrowFileSystem=` grows root to fill the USB on first
  boot -- the modern cloud/appliance-image idiom (this is what systemd-repart
  is actually for, versus the earlier, unworkable idea of carving an overlay
  into a live ISO).

### Compression: not a regression

A plain ext4 install would lose the squashfs/XZ compression the ISO had. The
fix is a compressing read-write root:

- btrfs with `compress=zstd` -- the Fedora Workstation / openSUSE default root
  fs -- gives transparent per-file zstd on a fully read-write filesystem.
  Composes with the stack: disk -> LUKS -> btrfs(zstd), compression before
  encryption; `grub-live`'s RAM overlay is fs-agnostic.
- f2fs with `compress_algorithm=zstd` -- same idea, flash-optimized.

btrfs-zstd does not match squashfs-XZ's ratio (XZ read-only large-block is
more aggressive), so the running footprint is larger than the squashfs image
-- but far smaller than uncompressed ext4, and read-write, which squashfs
never is. Download size is kept by shipping `.img.xz` (free space compresses
away); the "extra decompress step" is avoided because mainstream flashers
(balenaEtcher, Raspberry Pi Imager, GNOME Disks, `bmaptool` with a `.bmap`)
decompress on the fly during flash.

### Full-disk-encryption fork

A mass-produced image cannot bake a per-user passphrase. Two ways to reach
per-user FDE, asymmetric because of how cryptsetup works:

- Keyslot change (`luksAddKey`/`luksChangeKey`): online and instant -- works
  on the live mounted root.
- Data reencryption (`cryptsetup reencrypt --encrypt`/`--decrypt`): offline
  and slow -- cannot run on the mounted root; rewrites the data.

Option A (recommended) -- ship the root already inside LUKS2 with a
known/default keyslot that the initramfs unlocks automatically; FORCE the user
to set their own passphrase at first boot (a fast, online `luksAddKey` +
`luksRemoveKey` of the default). Encrypted from the first byte, no plaintext
window, fast first boot, user passphrase enrolled before any user data exists.
"Keep unencrypted" stays available as an explicit opt-out that runs the slow
offline `reencrypt --decrypt`.

Option B -- ship plaintext, opt into encryption at first boot via a slow
offline `reencrypt --encrypt`. Matches "unencrypted is the shipped state"
literally, but encryption is the slow path and there is a plaintext window.

Encrypted-`/home`-only (the Fedora `home.img` parallel) was rejected: the
requirement is full-disk encryption, not only `/home`.

### Amnesic mode

`grub-live` alone covers the amnesic use case (confirmed working on a
Calamares FDE install), so no amnesic-first artifact is required. A genuine
"convert to a clean FDE install" path remains available via `toram` +
Calamares from the ISO, separate from this feature.

### Artifacts

- Ship the pre-installed `.img` (`.img.xz` + `.bmap`) for the persistent USB.
- Keep the ISO for live/try/installer use, at minimal effort (candidate for
  later deprecation).


## Open questions / scoping

- Confirm derivative-maker can emit this layout: a btrfs-zstd (or f2fs-zstd)
  root image with a default LUKS keyslot, plaintext `/boot`, a generic
  initramfs, and first-boot rekey + grow units. (dev1116's persistence-setup
  already handled a btrfs option, so the tooling is not foreign.)
- Settle the FDE fork (A vs B).
- Verify first-boot rekey + grow interplay with Secure Boot and with
  `grub-live` on the shipped image.
