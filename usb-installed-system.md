# Persistent storage on a bootable USB

How do you give a USB you boot from whole-root persistence -- ideally with
full-disk encryption -- on the SAME medium the system booted from? This note
surveys how the major projects solve it, identifies the one constraint that
rules out the naive approach, and weighs the delivery options for
Kicksecure/Whonix. NOTHING is decided yet -- this records the leading candidate
and the tradeoffs, not a final choice.

A naming note up front: "persistent live USB" is a contradiction in terms. A
LIVE system is amnesic by design -- it runs from read-only media and discards
state at shutdown -- so it cannot also be persistent. The survey below shows
every genuine live-overlay route is either blocked (the whole-disk iso9660
constraint) or a bolt-on, and the leading (appliance) design resolves the
contradiction by dropping "live" entirely: it ships a normal, persistent install
on the USB and
lets `grub-live` provide amnesic-on-demand as an opt-in mode. So this is a
bootable/installed USB system, NOT a live USB -- which is why this note is named
`usb-installed-system`, not "live-usb-persistence".

Scope: Kicksecure/Whonix use dracut `dmsquash-live` with an overlayfs union
root. The findings generalize, but the leaning at the end is specific to
that stack.


## Goals (and where they conflict)

Wanted properties, collected so the impossible combinations are explicit. They
cannot all hold on one artifact; the conflicts are the point of listing them.

- G1 -- Persistent whole root that behaves like a normal install (logs, state,
  packages, home all survive reboot).
- G2 -- Full-disk encryption of the whole root (not only `/home`) available;
  the default is a CHOICE, not a silent on-or-off -- first boot asks the user
  to encrypt or not, rather than deciding for them.
- G3 -- Optional second disk as extra persistence via the same label.
- G4 -- Amnesic ON DEMAND (`grub-live`), not amnesic by default.
- G5 -- Easy to test in VirtualBox via the flow most users already know: attach
  an ISO to the virtual DVD/optical device and boot it.
- G6 -- Small download: high compression of the shipped artifact (squashfs/XZ
  is the strongest option).
- G7 -- Usability: NO extra user step from compression. The artifact must flash
  or boot with the tool the user already uses, with no manual `xz -d` unwrap
  first. This is not anti-compression -- it is pro-usability: compression is
  wanted, but a wrapper the user has to unpack by hand is the thing to avoid. If
  the flashing/boot tool read xz natively (balenaEtcher and Raspberry Pi Imager
  do; `dd` and `VBoxManage convertfromraw` do not), `.img.xz` would satisfy G6
  and G7 at once.
- G8 -- Simple partition design: as few partitions as possible, standard layout,
  no dedicated/auxiliary partitions (e.g. no separate LUKS-header partition). A
  detached-header FDE scheme is rejected on this ground (see the FDE fork).

### The conflicts

- G5 vs G1/G2 -- IMPOSSIBLE as stated. A virtual DVD is a READ-ONLY optical
  device and an ISO is read-only content: nothing booted from it can persist
  back to it, and a writable LUKS/FDE root cannot live on it. So "boot the
  persistent FDE image from the DVD/ISO device" cannot be done. The appliance
  `.img` is a DISK image -- in VirtualBox it is attached as a virtual HARD DISK
  (a one-time `VBoxManage convertfromraw image.img image.vdi`, or a raw-VMDK
  descriptor, then boot), NOT placed on the DVD device. Slightly less familiar
  than ISO-on-DVD, but a single documented command. The ISO-on-DVD flow stays
  available only for the LIVE/try/installer ISO (amnesic, or Calamares-installs
  to a separate virtual disk) -- which is not the persistent artifact.
  - Installer-test safety, a point FOR the ISO: booting the Calamares installer
    from a virtual DVD drive keeps the install SOURCE (the read-only optical
    ISO) physically distinct from the install TARGET (a blank virtual hard
    disk). Calamares cannot offer the read-only DVD as a destination, so the
    user cannot accidentally wipe or install onto the installer medium itself.
    If the installer instead shipped as a VDI/VMDK/OVA (a virtual HARD DISK),
    the installer medium and the target would both appear as disks in
    Calamares' partition step, inviting a mistaken pick of the installer disk.
    So the ISO/virtual-DVD medium is the safer, more familiar way to TEST the
    Calamares installer -- a reason to keep the installer path on the ISO,
    separate from the pre-installed persistent `.img`.

- G6 (squashfs) vs G1/G2 -- IMPOSSIBLE as a single root. squashfs is read-only
  and immutable; it can be neither a writable persistent root nor a writable
  LUKS FDE root. squashfs-as-root is the same live-vs-persistent contradiction
  restated at the filesystem layer.
  - Resolution proposed (see Compression below): compress the DOWNLOAD
    (`.img.xz` + `.bmap`, decompressed on the fly while flashing / before
    `convertfromraw`), run on a read-WRITE btrfs/f2fs `compress=zstd` root --
    compressed and persistent, larger than squashfs/XZ but far smaller than raw
    ext4.
  - The only way to KEEP squashfs AND persistence AND FDE is the verity-sealed
    split-root model (see `verified-measured-ima-boot.md`): a read-only,
    max-compressed squashfs/erofs `/usr` plus a writable LUKS `/var`+`/etc`+
    `/home`. One such image can back BOTH an ISO (DVD-bootable; writes ->
    tmpfs) and a disk image (writes -> persistent LUKS) -- the only way to get
    G5 + G6 together with persistence, at the cost of no longer being a plain
    "normal install."

- G6 vs G7 -- in tension for a raw disk image. A raw `.img` is zero-extra-step
  but large (it is the full partition size, not the used size; zstd compresses
  the data at REST inside the fs, but the file a user downloads is still the
  whole partition unless the FILE itself is compressed). Shrinking the DOWNLOAD
  means compressing the file (`.img.xz`), which is transparent only to flashers
  with native xz support; plain `dd` and VirtualBox's `convertfromraw` (G5) then
  need a manual `xz -d` first -- the extra step G7 rejects. The read-only-medium
  artifacts dodge this cleanly: a squashfs-based ISO ships compressed AND boots
  directly from the DVD device with no decompress step, satisfying G5 + G6 + G7
  at once -- but only amnesic, no G1/G2. So "small download AND zero extra step
  AND persistence" again points at the verity-split-root model (a compressed
  read-only `/usr` shipped as-is plus writable LUKS state), not a raw
  ext4/btrfs `.img`.
  - Two refinements keep this honest. First, the compression win is SQUASHFS
    (or erofs), not the ISO: squashfs can sit on a PARTITION of a normal GPT
    disk image with a writable partition beside it, so "stuck with squashfs"
    does NOT mean "stuck with an ISO." What forces read-only is the raw-image
    problem (a raw `.img` of a read-write fs is big-or-wrapped), not the
    container. Second, "zero extra step" is TOOL-RELATIVE, so the strict "one
    universal image, every tool including `dd`" rule is the only thing that
    truly forces a compressed read-only fs:
    - any tool incl. raw `dd`, one physical-USB image, persistent -> only a
      compressed read-only fs (squashfs/erofs) is both small and consumed
      as-is -> read-only -> live or verity-split (shippable as a GPT `.img`,
      not necessarily an ISO);
    - accept a mainstream flasher (balenaEtcher, Raspberry Pi Imager, GNOME
      Disks) -> `.img.xz` of a read-write btrfs/f2fs-zstd root is small AND
      zero-step AND persistent (what Raspberry Pi OS / Ubuntu / Fedora ship);
    - VirtualBox test path (G5) -> a compressed VMDK/VDI (or an OVA;
      streamOptimized VMDK is zlib-compressed) is small and boots directly, no
      squashfs, no ISO;
    - QEMU/KVM -> a compressed qcow2 boots directly.

Net: the "read-only medium" goals (G5 ISO-on-DVD, G6 squashfs-root) and the
zero-extra-step usability goal (G7) are mutually exclusive with the
plain-normal-persistent-FDE-install appliance candidate below, UNLESS the
design moves to the verity-split-root model, which reconciles them but is not a
plain install.


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
  (`70dmsquash-live-autooverlay`) skips on `[ "$partition" != "1" ] || [
  "$readonly" != "0" ]` ("Skipping overlay creation: unpartitioned or read-only
  media detected") -- it keys on the sysfs PARTITION INDEX and the read-only
  flag, NOT on any iso9660 detection. A raw-dd isohybrid is skipped by the
  PARTITION test: the live iso9660 label sits on the WHOLE disk, which has no
  sysfs partition index, so `partition=0 != 1`. It is NOT the read-only flag that
  fires -- `/sys/class/block/<dev>/ro` is the block-device flag and reads `0` on a
  writable USB, independent of the iso9660 content (confirmed empirically). Two
  edge cases follow from the literal test: a writable live fs that is not
  partition 1 is ALSO skipped, and a writable isohybrid exposed as partition 1
  would NOT be skipped. So the module catches the whole-disk iso9660 case via the
  partition half (no partition index), not via read-only.

A confirmed-working but non-idiomatic workaround exists (see "Loop-device
indirection"), but every project that actually ships persistence avoids the
problem instead of working around it: none of them raw-dd a whole-disk
isohybrid when persistence is wanted.


## How the major projects solve it

| Project | Medium layout | Persistence unit | Encryption | Provisioned |
|---|---|---|---|---|
| Tails (USB image `.img`, the persistence-capable format) | raw-written to the whole device, but a PARTITIONED GPT image -- an ESP (FAT, label `TAILS`) plus reserved free space, so the live fs is on a partition and the whole device is not iso9660 | "Persistent Storage": a LUKS/dm-crypt partition carved in the free space | mandatory LUKS (dm-crypt) | at runtime, by a wizard inside the booted system |
| Debian live-boot | user's choice: file on a writable fs, or a labeled partition | overlay FILE (`persistence` + `persistence.conf`) or partition | native `persistence-encryption=luks` | at prep time; discovered at boot by label/file |
| Fedora `livecd-iso-to-disk` | installer partitions + copies files (ext4/vfat/...), not dd | overlay FILE `/LiveOS/overlay-*` (dm-snapshot) or `--overlayfs` dirs | only `/home` (the separate `/LiveOS/home.img`) is LUKS by default; the root persistence overlay (changes to `/`) is unencrypted | at install time |
| Ubuntu casper | SDC/mkusb copy files to a FAT partition (+ a partition for the writable area) | `casper-rw`/`writable` FILE (<=4 GB) or labeled partition | none native (DIY LUKS) | at install time; discovered at boot by label |
| dracut `dmsquash-live` | whole-disk iso9660 for a raw-dd ISO; a partition/file on writable media otherwise | overlay FILE (`rd.overlay=LABEL=x:/path`, `:auto` -> `/LiveOS/overlay-<label>-<uuid>`) or directory | via `rd.luks.*` on the backing device | overlay file auto-created at boot only on writable media |

The common thread: three of the four shipping projects deliberately do NOT
put an iso9660 on the whole disk. They either (a) keep persistence as a FILE
on an already-writable filesystem, or (b) create the medium as a PARTITIONED
device via an installer (file-copy, not raw dd) so the live system lives on a
partition and the whole disk is never mounted.


### Tails -- LUKS partition, runtime wizard

Tails ships TWO formats, and only one supports persistence -- so it matters
which is meant:

- The USB IMAGE (`tails-amd64-<ver>.img`) is written RAW to the whole USB
  device (`dd`, GNOME Disks, balenaEtcher) -- but it is a PARTITIONED GPT
  image: an EFI System Partition (FAT, label `TAILS`) plus the rest of the
  device left as free space. Because the live system sits on a PARTITION, the
  whole block device is NOT mounted as one iso9660 filesystem, so the O_EXCL
  constraint above does not apply. The Persistent Storage wizard, run from
  inside the booted system, carves a LUKS/dm-crypt partition into that free
  space (called "Persistent Storage"; the older `TailsData` name predates the
  current wizard). Runtime partitioning succeeds precisely because the system
  is on a partition, not the whole disk.
- The ISO IMAGE (`tails-amd64-<ver>.iso`) is the whole-disk isohybrid, for
  burning a DVD or booting a VM. Tails explicitly directs USB users to the USB
  image instead. A DVD is read-only and a VM booting the read-only ISO has no
  writable medium, so Persistent Storage is not available from the ISO.

The directly-written USB image replaced the old (pre-3.12, before January
2019) two-step method -- boot an intermediary Tails, then run Tails Installer
(later renamed Tails Cloner) to COPY files onto the final stick. "Files copied,
not a raw dd" describes that legacy path, not today's USB image.

- https://tails.net/doc/persistent_storage/
- https://tails.net/install/download/ (USB image) ; https://tails.net/install/download-iso/ (ISO image: DVD/VM)
- https://tails.net/news/test_usb_images (3.12 directly-written USB image)
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
LUKS-encrypted by default; the persistence overlay that backs the rest of the
root filesystem (the `/LiveOS/overlay-*` CoW file or the `--overlayfs`
upper/work dirs described above) is unencrypted by default.
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

3. Tails model (the USB image, not the ISO) -- partitioned GPT medium +
   runtime LUKS partition carve.
   Idiomatic for amnesic-by-default with opt-in encrypted persistence. The
   whole disk is never mounted, so the carve works with no workaround. It
   matches the trailing-free-space LAYOUT requirement (a LUKS partition carved
   into reserved free space, same-label for a second disk) -- though Tails's own
   Persistent Storage is SELECTIVE (a per-feature list), not whole-root
   persistence; whole-root is Kicksecure's requirement, not Tails's behaviour.

4. Fedora model -- overlay FILE on a writable medium. Idiomatic for a
   convenience persistence feature. No repartitioning. But the classic
   dm-snapshot file has an overflow-to-read-only failure mode, the root
   overlay is unencrypted by default, and a file maps poorly to "trailing free
   space" and "a second whole disk via the same label".

5. dracut autooverlay -- boot-time new-partition carve, but gated to writable
   partitioned non-iso9660 media; confirms the constraint rather than solving
   it for raw-dd.

6. Appliance model -- ship a pre-installed normal disk image. See below. This
   is the current front-runner (nothing is decided yet).

7. Self-inflating squashfs `.img` -- ship a PLAIN `.img` (dd-compatible, no
   outer `.xz`) that is small because its rootfs is a squashfs INSIDE the image;
   on first boot the initramfs (or a first-boot service) unpacks the squashfs
   onto a plain rw root, deletes the squashfs, and grows root to fill the
   device. Tempting because it hits all of small-download + plain-img +
   zero-decompress-step + dd-compatible + normal-persistent-root at once. VERDICT:
   viable but NOT recommended, and effectively un-trodden -- see "Self-inflating
   squashfs" below.


## Leading candidate (not yet decided): the appliance model

Requirements that shape the leaning toward it: whole-root persistence that behaves like
a normal install; full-disk encryption (not just `/home`) available, with the
encrypt-or-not decision left to the user as an explicit first-boot CHOICE (not
silently defaulted on or off); a second disk as an option; and -- a late
reframing -- amnesic boot is NOT required as the default, because `grub-live`
already provides amnesic-on-demand from the GRUB menu on any installed system.

Given that, the live-overlay models (3, 4) are the wrong frame. The appliance
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

- ESP + plaintext `/boot` + LUKS2/Argon2id root. For legacy-BIOS boot the image
  also needs MBR boot code + a BIOS boot partition (`bios_grub`/`EF02`); an ESP
  alone is UEFI-only.
- Generic initramfs (`hostonly=no`) so the USB boots on any machine.
- `systemd-repart` grows the root PARTITION to fill the USB on first boot -- the
  modern cloud/appliance-image idiom. Growing the FILESYSTEM inside it is a
  separate step: the `GrowFileSystem=` flag is honored only by gpt-auto
  (Discoverable Partitions) mounts, NOT by explicit `fstab`/`crypttab` mounts, so
  an explicit-mount Debian layout needs a `systemd-growfs`/`resize2fs` step --
  and `cryptsetup resize` first for the LUKS mapping.

### Compression: not a regression

A plain ext4 install would lose the squashfs/XZ compression the ISO had. The
fix is a compressing read-write root:

- btrfs with `compress=zstd` -- the Fedora Workstation default root fs (btrfs
  `compress=zstd:1` since Fedora 34; openSUSE uses btrfs but does NOT enable
  compression by default) -- gives transparent per-file zstd on a fully
  read-write filesystem. Composes with the stack: disk -> LUKS -> btrfs(zstd),
  compression before encryption; `grub-live`'s RAM overlay is fs-agnostic.
- f2fs with `compress_algorithm=zstd` -- same idea, flash-optimized, but f2fs
  compresses only files flagged for it (`chattr +c`, or a `compress_extension=`
  mount option; `compress_extension=*` for all), not every file automatically.

btrfs-zstd does not match squashfs-XZ's ratio (XZ read-only large-block is
more aggressive), so the running footprint is larger than the squashfs image
-- but far smaller than uncompressed ext4, and read-write, which squashfs
never is. Download size is kept by shipping `.img.xz` (free space compresses
away). The "extra decompress step" (G7) is avoided ONLY for flashers that read
xz natively -- balenaEtcher and Raspberry Pi Imager decompress on the fly, and
GNOME Disks "Restore Disk Image" and `bmaptool` with a `.bmap` handle an xz
source too. It is NOT avoided for plain `dd` or for VirtualBox's
`VBoxManage convertfromraw` (G5's path), both of which need an uncompressed
`.img` and so force a manual `xz -d` first. So `.img.xz` trades download size
against a one-command unwrap for exactly the raw-`dd`/VirtualBox users; whether
that is acceptable is the open G6-vs-G7 call, not a solved "no regression." A
truly zero-step compressed artifact exists only on the read-only-medium side (a
squashfs ISO boots compressed with no unwrap), which cannot persist -- see the
Goals conflicts.

### Flashing an `.img.xz`: which tools decompress on the fly

Whether `.img.xz` is zero-step (G7) depends entirely on the flashing tool. The
ones that decompress xz WHILE writing to the device (no manual `xz -d`):

- balenaEtcher -- GUI; xz, gz, bz2, zip (not zstd). NOT in the Debian archive
  (balena ships an external `.deb`/AppImage only). Per-format decoders in its
  SDK: https://github.com/balena-io-modules/etcher-sdk/tree/master/lib/source-destination
- Raspberry Pi Imager -- GUI; libarchive-based, so xz/gz/bz2/zstd/zip. NOT in
  Debian (upstream/snap only). Source:
  https://github.com/raspberrypi/rpi-imager/blob/main/src/downloadextractthread.cpp
- GNOME Disks ("Restore Disk Image...") -- GUI; xz ONLY (gz/bz2/zst must be
  unpacked by hand, per the still-open request). IN Debian:
  `gnome-disk-utility` (trixie 46.1, bookworm 43.0).
  https://gitlab.gnome.org/GNOME/gnome-disk-utility/-/issues/342
- bmaptool -- CLI; decompresses xz/gz/bz2/zst/lz4/lzo on the fly. It uses a
  `.bmap` beside the image if present; with NO `.bmap` the bare `bmaptool copy`
  ERRORS ("bmap file not found, use --nobmap") and writes nothing, so you must
  pass `--nobmap` (`bmaptool copy --nobmap image.img.xz /dev/sdX`) to flash
  without bmap acceleration. This is effectively the "dd that understands xz." IN Debian: `bmap-tools` (bookworm 3.6-2); in trixie
  `bmap-tools` is a transitional package and the binary moved to the renamed
  `bmaptool` 3.9.0-3 (command is `bmaptool` either way).
  https://manpages.debian.org/trixie/bmaptool/bmaptool.1.en.html

So within the Debian archive itself -- the relevant ecosystem for a
Debian-based Kicksecure/Whonix -- the xz-decompressing writers are bmaptool
(CLI) and GNOME Disks (GUI, xz-only). balenaEtcher, Raspberry Pi Imager and
usbimager are NOT packaged (usbimager and the GNOME GUI Impression do xz
upstream but are absent from Debian stable; Impression is testing/sid only),
and `gnome-multi-writer` is packaged but writes raw with no decompression.

Zero-install fallback, always available (`dd` from coreutils + `xzcat` from
xz-utils, every suite):

    xzcat image.img.xz | dd of=/dev/sdX bs=4M status=progress

NOT decompressed by: plain `dd` alone, and VirtualBox's
`VBoxManage convertfromraw` (the G5 path), which needs an uncompressed raw
image -- so `.img.xz` still costs those users a manual `xz -d`.
https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/vboxmanage-convertfromraw.html

### Self-inflating squashfs (approach 7): tempting, not recommended

The idea: ship a PLAIN `.img` (no outer `.xz`, dd-compatible) that is small
because its rootfs is a squashfs stored INSIDE the image; on first boot, unpack
the squashfs onto a plain read-write root, delete the squashfs, and grow root to
fill the device. On paper it satisfies G1 + G2 + G6 + G7 together, which nothing
else does (not G5 -- a plain `.img` is not the ISO-on-DVD familiar flow; that
stays the "middle" VBox path). In practice it is viable but carries real cost,
and -- a strong signal -- no mainstream project actually ships it.

- NO PRIOR ART for the exact pattern (a squashfs unpacked IN PLACE to a plain rw
  root on one self-inflating medium, squashfs then discarded). The closest are
  block-level, from SEPARATE install media: SUSE KIWI OEM (`dd`-dumps a
  compressed `.raw` disk image to the target, then `oem-resize` grows root) and
  Clonezilla/partclone (compressed used-block fs-image restore + resize).
  systemd-repart `CopyBlocks=auto` is the recognized "self-replicating image"
  primitive but is RAW-block, not a compressed unpack. dracut `dmsquash-live`
  keeps squashfs READ-ONLY with an overlay -- it never inflates to a plain root.
  cloud-init / rpi `init_resize` / postmarketOS only GROW an already-plain fs;
  that is not inflation. (Refs: systemd-repart.8, repart.d.5; KIWI
  dracut-kiwi-oem-dump; squashfs-tools-ng benchmark; sigma-star erofs-vs-squashfs.)
- MECHANISM MISMATCH is the crux: `unsquashfs` extracts a directory TREE to a
  directory, not to a block device, so the flow must be mkfs -> mount ->
  `unsquashfs -d` -> unmount. squashfs is a filesystem container, not a block
  image, so it slots into NO stock block-level deployment primitive (dd,
  `CopyBlocks=`, partclone). You hand-roll and maintain a bespoke dracut module
  or first-boot service. systemd-repart only grows/adds partitions (never
  shrinks/moves/deletes outside factory-reset), so the squashfs must live in a
  TRAILING partition and the ordering is awkward: the root partition has to be
  shipped already large enough to hold the inflated tree (you cannot grow root
  into the trailing space until the squashfs is deleted, yet you need the
  squashfs present to inflate FROM) -- so it is inflate into a pre-sized root,
  then delete the trailing squashfs partition, then grow to fill the rest. That
  "ship root pre-sized to the full inflated size" requirement fights the
  small-image goal and is part of why this is brittle.
- FIRST-BOOT TIME is paid on the user's (often weak) device, not once on a build
  host. Estimate for ~2 GB compressed -> ~6-8 GB root: NVMe/SATA-SSD with zstd
  ~15-60 s; a good USB 3.0 stick ~2-4 min; a cheap USB 3.0 stick ~5-10 min. On
  fast storage xz decompression is the bottleneck (prefer zstd/lz4); on slow USB
  the write dominates and the compressor barely matters.
- ROBUSTNESS REGRESSION: a power loss mid-inflate bricks first boot unless the
  inflate is engineered idempotent/resumable -- worse than a plain flashed image,
  which is atomic.
- FDE/Secure Boot: LUKS is fine (inflate runs after unlock, into the encrypted
  container; the staging squashfs is public plaintext). But embedding the
  inflate tooling in the initramfs bloats it and, under a signed UKI, forces a
  re-sign and changes measurements; a first-boot service avoids touching signed
  artifacts but is itself unprotected.

VERDICT: not recommended. The `.img.xz` + xz-aware flasher path (previous two
sections) already delivers small download + plain persistent root with NO
first-boot penalty and NO bespoke initramfs -- it decompresses on the build/flash
host (fast, free, atomic) instead of on the slow target. The only property it
lacks is that the STORED file is itself a bare dd-able image, and mainstream
flashers eat `.xz` transparently, so that gap is largely cosmetic. Self-inflation
trades a one-time host-side decompress for a per-user first-boot decompress plus
a bespoke, brittle module -- a worse trade for this feature. (It is a genuinely
different thing from verity-split-root, which keeps root immutable by design and
so is not a substitute here.)

### Full-disk-encryption fork

A mass-produced image cannot bake a per-user passphrase. Two ways to reach
per-user FDE, asymmetric because of how cryptsetup works:

- Keyslot change (`luksAddKey`/`luksChangeKey`): online and instant -- works
  on the live mounted root.
- Data reencryption (`cryptsetup reencrypt --encrypt`/`--decrypt`): slow,
  rewrites the data. LUKS2 supports ONLINE reencryption (cryptsetup 2.2.0+) for a
  cipher/key change on a mounted device, but ADDING or REMOVING the dm-crypt layer
  (`--encrypt`/`--decrypt`) still needs an offline window you cannot take on the
  root you are booted from -- so run it from the initramfs/live environment, not
  the running root.

Per G2 the first boot PRESENTS the encrypt-or-not choice either way. A and B
differ in the shipped starting state (A ships encrypted under a public default
key; B ships plaintext), but BOTH need a first-boot `reencrypt` to reach real
per-user confidentiality -- A to rotate off the public master key, B to encrypt
the plaintext -- so the earlier "A is the fast keyslot path" framing does not hold.

Option A -- ship the root already inside LUKS2 with a known/default keyslot the
initramfs unlocks automatically, then convert at first boot. The subtlety that
sinks the naive version: `luksAddKey`/`luksRemoveKey` change only KEYSLOTS, never
the master (volume) key. That master key is baked at build time and is PUBLIC --
anyone with the image recovers it by unlocking the shipped header with the
default passphrase -- so removing the default keyslot does NOT make the volume
confidential: the data stays decryptable with the public master key (directly via
`cryptsetup open --master-key-file`, or by restoring the shipped header), and a
user who merely ADDS a passphrase writes all later data under that same public
key. Real confidentiality requires ROTATING the master key, which only
`cryptsetup reencrypt` does -- a first-boot full `reencrypt` with a fresh volume
key, not a keyslot swap. Done before user data and before the grow (on the small
pre-grow image) it is bounded (minutes) and crash-resumable. To avoid the
destructive `--reduce-device-size` datashift, build the image with the ~16-32 MiB
LUKS2 header gap PRE-RESERVED at the front of the root partition -- this keeps the
standard integrated-header layout (`luksOpen /dev/root`, `crypttab` and recovery
work with no `header=` plumbing) and only de-risks the shift (it moves into
reserved empty space). A detached header would avoid the datashift entirely but
needs a dedicated header partition, which conflicts with the simple-partition
goal (G8), so the pre-reserved integrated gap is the choice here. Prior art:
SUSE's `disk-encryption-tool` does a first-boot in-place `reencrypt --encrypt` on
its Micro appliance images. "Keep unencrypted" stays an explicit opt-out
(`reencrypt --decrypt`).

Disadvantage of A (encrypted by default), weighed against G2's "simple to
mount with standard commands": the shipped image is a PARTITIONED disk image, so
neither option mounts with a bare `mount -o loop image.img /mnt` -- both need the
partitions mapped first (`losetup -P image.img`, or `kpartx -a`) and then the
root partition mounted. The difference is one step: Option A additionally
requires `cryptsetup luksOpen` of the mapped root partition (with the shipped
default passphrase) before it can be mounted, for EVERY user including one who
does not want encryption. So inspection, recovery, and scripted provisioning gain
that extra cryptsetup step under A. B needs the same partition mapping but no
cryptsetup, at the price of a plaintext shipped state. This is a real usability
cost of A, not a security one, and is documented here as the tradeoff rather than
resolved.

Option B -- ship plaintext, opt into encryption at first boot via a slow
offline `reencrypt --encrypt`. Matches "unencrypted is the shipped state"
literally, but encryption is the slow path and there is a plaintext window.

Encrypted-`/home`-only (the Fedora `home.img` parallel) was rejected: the
requirement is full-disk encryption, not only `/home`.

### Known limitation: unencrypted `/boot` is a tamper surface

FDE on the root leaves `/boot` (and the ESP, `grub.cfg`, and initramfs)
outside LUKS -- it must stay readable before the root is unlocked. A physical
attacker with the stick can therefore tamper with the boot chain: edit the
kernel command line (`init=/bin/sh`), or swap the initramfs to capture the
LUKS passphrase at the next unlock. Debian's signed shim+GRUB verifies the
kernel signature but NOT the initramfs or the `grub.cfg` body, so code signing
alone does not close it. This is the same evil-maid surface every standard
unencrypted-`/boot` FDE install has, accepted here for now ("full FDE except
`/boot`"). The real mitigation is a measured/sealed boot chain (Secure Boot +
TPM measurement, or a signed Unified Kernel Image), not a code change -- a
candidate for a later iteration, documented as a known limitation until then.

### Amnesic mode

`grub-live` alone covers the amnesic use case (confirmed working on a
Calamares FDE install), so no amnesic-first artifact is required. A genuine
"convert to a clean FDE install" path remains available via `toram` +
Calamares from the ISO, separate from this feature.

### Testing in VirtualBox

Note on "no `convertfromraw`": a flat-VMDK descriptor is a ~300-byte TEXT sidecar
whose extent line is `RW <sectors> FLAT "image.raw" 0` -- it references the raw
`.img` IN PLACE (the raw bytes are the storage), so it is NOT a conversion and
copies nothing. `convertfromraw`, by contrast, writes a whole new VDI/VMDK with
the data copied. VBox will not attach a completely bare `.img` as a disk, so the
descriptor is the minimal zero-copy glue; booting through it IS booting the raw
image. (QEMU needs no glue at all: `-drive file=image.img,format=raw`.)

Three ways to test, from least to most confusable about which disk the installer
targets:

1. ISO on the virtual DVD drive (BEST, and the familiar flow). The source is a
   read-only optical device; Calamares cannot offer it as a destination, so the
   wrong-target mistake is impossible. This is the installer/try path (the ISO
   artifact), not the persistent `.img`.
2. The `.img` as a virtual USB/removable device -- TESTED, does NOT work in
   VirtualBox 7.2.20. The intent (mirror a physical USB-boot, and keep the
   removable medium distinguishable from the fixed Calamares target) rested on
   the 7.x manual's "USB-attached drives boot under EFI". We ran it:
   - VirtualBox 7.x has a real USB storage controller; each disk attached to it
     "appears as a dedicated USB device to the guest." But: "Booting from drives
     attached using USB is only supported when EFI is used as the BIOS lacks USB
     support" (official 7.x manual, storage.html) -- so EFI firmware is
     mandatory for this path.
   - Attach the `.img` with NO data copy via a hand-written flat-extent VMDK
     descriptor (`createType="monolithicFlat"`, one `RW <sectors> FLAT
     "image.img" 0` extent) -- the descriptor references the `.img` in place. The
     documented `createmedium --variant RawDisk --property RawDrive=` is for
     PHYSICAL device nodes (`/dev/sdX`), not a file, so the flat descriptor is
     the file route.
   - Sketch (EFI on; USB controller bootable; blank VDI target on SATA):
         VBoxManage modifyvm   "TestVM" --firmware efi
         VBoxManage storagectl "TestVM" --name usb0  --add usb --bootable on
         VBoxManage storageattach "TestVM" --storagectl usb0 --port 0 --device 0 \
            --type hdd --medium /abs/image.vmdk --hotpluggable on
         VBoxManage storagectl "TestVM" --name sata0 --add sata
         VBoxManage storageattach "TestVM" --storagectl sata0 --port 0 --device 0 \
            --type hdd --medium /abs/target.vdi
     The `.img` then appears to the guest as removable USB; `target.vdi` as a
     fixed internal disk. (`--hotpluggable`/`--nonrotational` only set hot-plug
     and SSD hints -- they do NOT make a non-USB-controller disk removable; only
     attaching to the USB controller does.)
   - RESULT (TESTED, VBox 7.2.20, USB storage controller + `--usbxhci on`): the
     VM powered on but the EFI firmware enumerated ONLY the blank SATA disk and
     printed "No bootable option or device was found" -- it did NOT boot the
     USB-attached disk. We then entered the firmware by hand to rule out "just not
     auto-discovered": the EFI Boot Manager listed only the two PXE entries (no
     disk), and Boot Maintenance Manager -> Boot From File opened an EMPTY File
     Explorer (no filesystem at all). So the firmware does not enumerate a
     USB-storage-controller disk as a boot device OR a filesystem -- there is no
     manual-selection or persisted-entry path either. The IDENTICAL flat-VMDK on
     a SATA controller booted Kicksecure fully (see 3) and uses the same ESP, so
     this is a USB-path limitation, not an image problem. The manual's
     USB-boot-under-EFI claim did not hold in 7.2.20: treat VBox USB-device boot
     as NON-WORKING (use the SATA flat-VMDK in 3 instead). Corroborated by a VBox
     forum moderator (forums.virtualbox.org thread t=87303): "Neither the BIOS nor
     the EFI implementations allow for a USB controller based disk to boot" (same
     setup -- a disk on the usb storage controller, invisible to EFI, booting fine
     on SATA). The manual's "USB-attached drives boot under EFI" line apparently
     means physical USB passthrough (rawdisk), not a disk on the usb storage
     controller; no on-point bugtracker ticket documents it.
   - Can we EMULATE physical USB passthrough so EFI boots it? TESTED, no. We
     built a removable USB mass-storage gadget on the host backed by the `.img`
     (Linux `dummy_hcd` virtual UDC + `g_mass_storage file=... removable=1`, both
     stock modules on the Debian/Kicksecure kernel) and captured it into the VM
     via a VBox USB device filter. VBox DID capture and attach it (guest showed
     "Currently attached USB devices: 0525:a4a5 ... Linux File-backed Storage
     Gadget" on xHCI) -- yet EFI STILL showed an empty Boot Manager and empty
     File Explorer and "No bootable option or device was found". So VBox 7.2.20
     EFI does not enumerate/boot a USB mass-storage device even when genuinely
     passed through, not only on the usb storage controller. (This is NOT a
     licensing limit: USB 2.0/3.0 passthrough is in the Free base package since
     VBox 7.0.0 -- no Extension Pack -- so the "needs the proprietary Ext Pack"
     claim is stale 6.0 docs.) Residual caveat: a real physical stick was not
     testable on the headless server, but VBox presented the gadget to the guest
     as an ordinary xHCI USB MSC, which EFI enumerates independently of the host
     backing, so the limitation is very likely VBox EFI itself.
   - Can we plug a DIFFERENT EFI firmware in? Possible but no help. VBox loads its
     EFI from a swappable `.fd` (`VBoxInternal/Devices/efi/0/Config/EfiRom`
     extradata; 7.x ships `VBoxEFI-amd64.fd` as a file), but stock OVMF won't fix
     this: VBox's firmware already LOADS `UsbBusDxe`/`UsbMassStorageDxe` (so the
     mass-storage driver is NOT the gap), and upstream OVMF is tied to QEMU's
     platform (fw_cfg etc.) and won't even boot on VBox. Third-party reports
     (Unraid/Haiku/OpenWrt) say VBox EFI boots USB via EHCI (USB 2.0) but not xHCI
     (USB 3.0) -- so we re-tested the gadget with ONLY OHCI+EHCI (xHCI off): same
     result -- captured and attached, but EFI's Boot Manager and File Explorer
     stayed empty. So neither EHCI nor xHCI enumerated it here. (Confound: a
     dummy_hcd gadget plus VBox's late USB capture is not a physical stick present
     before power-on, which is what those reports used; untestable on a headless
     server.) Net: USB boot of the `.img` is not achievable in VBox by any route
     tested here -- use the SATA flat-VMDK (3), or QEMU (`-device usb-storage,
     bootindex=0` + real OVMF) if USB-boot realism is actually needed.
   - BIOS USB-boot workaround (for REAL hardware / QEMU -- NOT VBox, which did
     not USB-boot even under EFI, above): for an old machine whose BIOS cannot
     USB-boot, best first: (a) enable UEFI; (b) use QEMU instead of VirtualBox --
     its default SeaBIOS
     firmware (Free Software, Debian `seabios`) boots USB natively, no shim;
     (c) if it must be VBox BIOS, boot a GRUB2 (BIOS) rescue ISO from the
     virtual DVD and `insmod ehci uhci ohci usbms` then chainload the USB disk
     -- Free Software (`grub-pc-bin`) but BEST-EFFORT and KNOWN-FLAKY (GRUB's
     native USB stack conflicts with the BIOS hand-off; hangs are common,
     EHCI/USB-2 historically worst). PLoP Boot Manager is the classic
     convenient shim here BUT is proprietary freeware (no source, no-sale
     license, abandoned since 2013, not in Debian) -- so under the
     Free-Software-only policy it is at most an unofficial DEVELOPER test
     convenience, never shipped or recommended to users. iPXE and SYSLINUX
     MEMDISK do NOT help (no USB host-controller drivers); writing a bespoke
     real-mode USB boot manager is a person-years systems project (reuse
     GRUB2/SeaBIOS), not a worthwhile build.
3. The `.img` as a plain virtual HARD DISK (flat-VMDK on SATA) -- TESTED, WORKS
   under BOTH EFI and BIOS, with NO `convertfromraw`. Attach the same flat-extent
   VMDK descriptor to a SATA controller (`storageattach ... --type hdd --medium
   image.vmdk`); the raw `.img` is referenced in place, no data copy. Verified: a
   100 GiB Kicksecure-LXQt raw (GPT: ESP + `bios_grub` + root) booted to the LXQt
   desktop under `--firmware efi` AND `--firmware bios`. This is the working
   no-convert way to test the appliance `.img` in VBox. Downside (unchanged): it
   presents as a FIXED disk, indistinguishable from a blank target in Calamares,
   so the ISO-on-DVD path (1) stays safest for installer testing; the
   removable-USB distinguishability idea (2) is not achievable in VBox.

For the persistent appliance `.img` itself (booting it as a normal install, no
installer), the SATA flat-VMDK route (3) is the working attach -- no
`convertfromraw`, boots under EFI or BIOS. The USB-device route (2) did not boot
in VBox.

### Artifacts

Candidate artifacts (nothing decided):

- a pre-installed `.img` (`.img.xz` + `.bmap`) for the persistent USB;
- the ISO retained for live/try/installer use (candidate for later deprecation).


## Artifact comparison

The delivery artifacts considered, against the goals (G1-G7) and the properties
that decide between them. "choice" = first boot asks the user (G2). VM-test
formats (qcow2 / VMDK / OVA) are a separate testing convenience, not a user
delivery artifact -- see "Testing in VirtualBox".

Legend (GitHub-rendered emoji; the source stays ASCII shortcodes):
:white_check_mark: yes / strong, :x: no / absent, :warning: caveat / partial,
:heavy_minus_sign: not applicable.

| Feature | ISO (live + installer) | Appliance `.img.xz` | Self-inflating squashfs `.img` | Verity-split-root `.img` |
|---|---|---|---|---|
| Role | live/try + Calamares installer source | pre-installed normal persistent system on the USB | plain `.img` with a squashfs inside that inflates to a rw root on first boot | read-only compressed `/usr` + writable LUKS state |
| Whole-root persistence (G1) | :x: no (amnesic; persists only after installing to a SEPARATE disk) | :white_check_mark: yes (a real install) | :white_check_mark: yes (after the first-boot inflate) | :warning: state persists, but the root is immutable (not a normal-install package model) |
| Root FDE, default=choice (G2) | :warning: only via the Calamares install-to-disk, not on the ISO | :white_check_mark: yes, whole-root LUKS; encrypt-or-not is a first-boot choice | :white_check_mark: yes (inflate into a LUKS container) | :warning: writable state is LUKS; `/usr` is public (verity integrity, not encrypted) |
| Download size (G6) | :white_check_mark: small (squashfs/XZ inside) | :warning: medium (btrfs/f2fs-zstd at rest, then `.xz`); bigger than squashfs/XZ | :white_check_mark: small (squashfs inside, no outer wrapper) | :white_check_mark: small (squashfs/erofs `/usr`) |
| Outer wrapper / decompress step (G7) | :white_check_mark: none -- boots and flashes compressed, zero step | :warning: `.xz` wrapper; zero-step only on xz-aware flashers, else a manual `xz -d` | :white_check_mark: none (plain dd-able `.img`, zero step) | :white_check_mark: can ship as a plain `.img` (compressed at rest), zero step |
| dd / flasher compatibility | :white_check_mark: dd-able isohybrid; DVD-burnable | :warning: xz-aware flashers write `.img.xz` inline (zero-step); plain `dd` needs `xz -d` first | :white_check_mark: dd-able | :white_check_mark: dd-able |
| Familiar VBox test (G5) | :white_check_mark: BEST -- attach to the virtual DVD and boot | :warning: middle -- attach as a virtual USB (EFI) or hard disk | :warning: middle -- as USB/HDD | :warning: middle -- one image can back BOTH an ISO and a disk |
| Calamares wrong-target safety | :white_check_mark: BEST -- read-only DVD source can never be the target | :heavy_minus_sign: n/a (no installer; boots directly) | :heavy_minus_sign: n/a (no installer) | :heavy_minus_sign: n/a (no installer) |
| Boot medium / rootfs | read-only whole-disk iso9660 + squashfs | writable GPT disk; btrfs/f2fs(zstd) root, plaintext `/boot`, LUKS root | writable disk; squashfs staging -> rw root | writable disk; dm-verity read-only `/usr` + writable state |
| Kernel upgrades | :warning: n/a while live; normal once installed | :white_check_mark: normal (new kernel lands in `/boot`, GRUB reads it) | :white_check_mark: normal (after inflate) | :warning: image-based (A/B or reseal), NOT plain apt-in-place |
| Amnesic | :white_check_mark: yes, by default | :white_check_mark: on demand via `grub-live` | :white_check_mark: on demand via `grub-live` | :white_check_mark: on demand via `grub-live`; also the ISO (writes -> tmpfs) |
| First-boot cost | :white_check_mark: none | :white_check_mark: fast (rekey + grow) | :warning: SLOW (unsquashfs: ~15-60 s on SSD, ~2-10 min on USB) | :white_check_mark: fast (grow) |
| Tooling maturity / new code | :white_check_mark: mature (exists today) | :warning: standard install layout + systemd-repart; modest new build tooling | :x: bespoke dracut/first-boot module; no prior art; brittle (power-loss mid-inflate) | :warning: more complex (verity/UKI/sealing); integrity bonus |
| Main advantage | familiar flow, zero-step compression, safest installer test | a real normal persistent FDE install; normal `apt`/kernel upgrades | small + plain + persistent all at once | small + zero-step + persistent + boot integrity |
| Main disadvantage | not persistent -- it is amnesic by nature | `.xz` step for raw-`dd`/VirtualBox users; larger than squashfs/XZ | slow, brittle first boot + bespoke code for a cosmetic gain | immutable root (no plain in-place `apt`); most complexity |

### ISO vs drive-image: delivery-format criteria

The criteria below weigh an ISO against ANY raw drive-image (our appliance
`.img`, self-inflating, and verity-split are all "drive-image" here). Each is
tagged: `fact` objective, `opinion` a maintainer/advocate view, `cp` a
counterpoint, `gap` an unresolved/unpublished question. Drawn from a survey of
the ISO-vs-drive-image debate.

| Criterion | ISO | Drive-image (`.img`) | Kind |
|---|---|---|---|
| Format lineage | ISO9660 (1988) + hybrid bolt-ons | native modern disk layout | fact + opinion |
| How it writes to USB | isohybrid is a dd-hack layered over iso9660 (bends the spec) | a real partitioned disk, dd'd natively | fact |
| Geometry/size honesty | partition table misstates CHS/GPT size | same ambiguity -- not actually cleaner | cp |
| USB write workflow | same tools + "boot from it" step | same tools + step | fact (neutral) |
| Partition table | a hybrid MBR/GPT IS present, but the iso9660 fs spans the whole device (not confined to a partition) | real GPT: ESP + `/boot` + root | fact |
| Space after writing | remainder is dead space | working partition grows to fill the drive (via a first-boot grow step, not automatically) | fact |
| Persistence out of the box | read-only; needs an overlay or extra partition | read-write; save area present at first boot | fact |
| Can it persist at all | yes (Knoppix-style overlay / extra writable partition) | yes, natively | cp (ISO can too) |
| Immutability | read-only by nature | writable | fact (cuts both ways) |
| Kernel/initrd update | rebuild the whole ISO | replace `vmlinuz` / swap initrd+sfs in place, or normal `apt` | fact |
| Edit files on the medium | no | yes (mount/click the `.img`) | fact |
| BIOS + UEFI boot | isohybrid serves both via hybrid cruft | ESP for UEFI; legacy BIOS needs MBR code + a `bios_grub` partition; single-stage menu | fact + opinion |
| dd-to-USB UEFI issues | can occur | avoided (native image) | cp (pro-image) |
| Secure Boot / signing | the surveyed sources are silent | silent -- OUR doc covers it (see Known limitation) | gap |
| Download form | `.iso` | `.img.gz`/`.img.xz` (decompress, or xz-aware flasher) | fact |
| Download size head-to-head | not published anywhere | not published anywhere | gap |
| Flashing-tool ecosystem | huge (many mainstream flashers + multiboot tools) | fewer image writers; some multiboot tools also write `.img` | cp (ISO breadth) |
| Re-verify vs published hash | yes, any time (read-only) | no -- it mutates on first boot (rekey/grow) | cp (pro-ISO) |
| Write verification | format-independent (the writer can checksum after write) | same | fact (neutral) |
| VM import | boots from the virtual DVD directly | often needs `.img` -> VDI/VMDK conversion | cp/fact (pro-ISO) |
| VM drive-mapping | predictable (DVD is not a disk) | as a disk it can confuse primary vs secondary | cp (pro-ISO) |
| Multiboot managers | native ISO ecosystem | multiboot managers can boot `.img` too; smaller ecosystem | cp (pro-ISO) |
| Remote/BMC virtual media (IPMI/iDRAC/iLO) | mounts an ISO -- the sysadmin standard | a raw `.img` is generally not a BMC media type | cp (pro-ISO, strong) |
| PXE / netinst | strong ecosystem | weaker for a raw image | cp (pro-ISO) |
| Optical / non-USB-bootable legacy PCs | CD-R still works | needs USB-boot or a frugal install | cp (pro-ISO, marginal) |
| Read-only clean environment (banking/forensics) | tamper-resistant | writable | cp (pro-ISO) |
| At-rest encryption | n/a (read-only) | fscrypt/LUKS on the working partition | fact (pro-image) |
| Producer maintenance | live-CD caveats pervade the docs | one coherent persistent model | opinion (pro-image) |
| Cost of shipping BOTH | an ISO build is cheap and well-trodden | -- | cp (dropping ISO is motivation, not a tech blocker) |
| Standards / anti-fragmentation / user choice | proven cross-platform standard | newer, less universal | cp (pro-ISO) |
| Host-OS inspect/mount of the container | Windows natively mounts/burns ISO | multi-partition `.img`/`.sfs` not easily opened on Windows (click-opens on Linux) | cp (pro-ISO) |
| Drive-image vs rootfs-image | -- | a PARTITIONED drive-image click-opens (N partitions); the "opaque/nested" gripe is about ROOTFS-images, not this | fact (clarifies pro-image) |
| Direct-file-boot (multiboot managers) | boots ISO | ALSO boots `.img`/VHD in practice -- not ISO-exclusive | fact (neutral) |
| ...but needs build-time support | initrd search code + loopback.conf baked by the creator | same -- a build-time choice, not a free container property | fact (neutral) |
| Frugal-install INSTALLER ecosystem | existing frugal-install + Windows dual-boot installers consume ISO | migrating them to `.img` = re-coding; also covers install-to-internal-drive + Win dual-boot | cp (pro-ISO, migration cost) |
| Multi-distro coexistence on one stick | many distros as frugal installs on one boot menu | `.img` wants the whole stick; reclaim/add is manual | cp (pro-ISO) |
| dd invocation footgun | burn/GUI tools hide it | exact `dd` command error-prone for newbies; they want a GUI installer | cp (pro-ISO) |
| Secure Boot cert copy fidelity | most ISO->USB tools OMIT the distro `.cer` (only a few keep it) | a writable image can carry it | cp (concrete SB-gap datapoint) |
| `.img.gz` loop-mount on tiny storage | ISO/SFS loop-mount as shipped | cannot mount a `.gz` without first writing the full uncompressed `.img`; a fix: a tiny ~8 MB 2nd partition so compressed ~= uncompressed, then ship an UNCOMPRESSED `.img` | fact + pro-image resolution |
| VM: QEMU vs VirtualBox | universal virtual optical drive | QEMU boots a raw `.img` directly (`-hda`, no convert); VirtualBox needs a VDI/VMDK wrapper | fact (QEMU easy, VBox friction) |
| Size-ceiling as anti-bloat discipline | CD/DVD ceiling nudges smaller builds | no inherent ceiling | opinion (pro-ISO, marginal) |
| Optical disc as archival hard-copy | durable physical recovery fallback | -- | opinion (pro-ISO; discs degrade, drives vanishing) |
| VM-trial / adoption barrier | easy VM trial via the virtual DVD | dropping ISO entirely blocks easy VM trial -> adoption/UX cost | opinion (pro-ISO) |
| Installed-base / network-effect inertia | the format the "many" already use | newer; must overcome inertia | opinion (pro-ISO) |
| Net direction | retain for install/try, VM, BMC/PXE, re-verifiable-hash, read-only use | leads for the PERSISTENT artifact (real install, whole-root FDE, normal upgrades); the two are complementary, not either/or | net |

Third-way formats also considered (neither a plain ISO nor a whole-disk `.img`):

- a single `.sfs` (kernel+initrd+sfs) -- easy to build (mksquashfs) and inspect,
  but no Windows host tooling;
- a ZIP / self-extracting `.exe` / tar -- extract-to-target = a frugal install
  (ZIP loses Unix ownership/permissions; tar keeps them);
- QCOW2 -- built-in encryption, dynamic grow, snapshots, native to QEMU;
- a tiny cross-platform download-script that fetches vmlinuz/initrd/sfs and adds
  a boot entry (no formatting, no image);
- a hybrid-ISO with an appended writable partition (keeps the ISO and gains
  persistence);
- an `img`<->`iso` converter so no producer is forced to ship both.


## Open questions / scoping

- Confirm derivative-maker can emit this layout: a btrfs-zstd (or f2fs-zstd)
  root image with a default LUKS keyslot, plaintext `/boot`, a generic
  initramfs, and first-boot rekey + grow units. (dev1116's persistence-setup
  already handled a btrfs option, so the tooling is not foreign.)
- Settle the FDE fork (A vs B).
- Verify first-boot rekey + grow interplay with Secure Boot and with
  `grub-live` on the shipped image.
