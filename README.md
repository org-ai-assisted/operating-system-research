# operating-system-research

Research notes on operating-system design questions: how comparable projects
solve a problem, what is idiomatic, and which design we chose and why.

Each document states the problem, surveys how real projects (named, with
citations) solve it, judges the options, and records the decision.

## Index

- [live-usb-persistence.md](live-usb-persistence.md) -- how to provide
  persistent storage (and full-disk encryption) on a live USB the system
  booted from. Surveys Tails, Debian live-boot, Fedora `livecd-iso-to-disk`,
  Ubuntu casper/mkusb, Ventoy, and dracut `dmsquash-live`; records the shift
  from a live-overlay model to a pre-installed appliance-image model for
  Kicksecure/Whonix.
- [verified-measured-ima-boot.md](verified-measured-ima-boot.md) -- verified
  boot, measured boot, and IMA for a Debian base (live ISO and installed),
  keeping GRUB and BIOS. Surveys systemd/ParticleOS, Fedora bootc/composefs,
  openSUSE MicroOS, Ubuntu Core, Arch/verity-squash-root, Android/ChromeOS,
  Flatcar, Azure Linux, Talos, Garden Linux; settles two facts empirically in a
  Secure Boot + TPM sandbox; shows stock Debian GRUB cannot confine boot to a
  signed UKI, and recommends shim -> systemd-boot -> signed UKI (own-signed GRUB
  only if GRUB on UEFI is mandatory), dm-verity roothash on the signed cmdline,
  IMA measurement-only for attestation.
