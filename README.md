# operating-system-research

Research notes on operating-system design questions: how comparable projects
solve a problem, what is idiomatic, and which design we chose and why.

Each document states the problem, surveys how real projects (named, with
citations) solve it, judges the options, and records the decision.

## Index

- [usb-installed-system.md](usb-installed-system.md) -- how to provide
  whole-root persistence (and full-disk encryption) on a USB you boot from.
  Surveys Tails, Debian live-boot, Fedora `livecd-iso-to-disk`, Ubuntu
  casper/mkusb, and dracut `dmsquash-live`; weighs a live-overlay model against
  a pre-installed appliance-image model for Kicksecure/Whonix (a normal install
  on a USB, not a live USB), leaning toward the appliance -- nothing decided yet.
- [verified-measured-ima-boot.md](verified-measured-ima-boot.md) -- verified,
  measured, and IMA boot for a Debian base (live ISO and installed) to a
  PREVENTION (Sovereign/Secure Boot) standard, keeping MOK and BIOS. Surveys
  systemd/ParticleOS, Fedora bootc/composefs, openSUSE MicroOS, Ubuntu Core,
  Arch/verity-squash-root, Android/ChromeOS, Flatcar, Azure Linux, Talos, Garden
  Linux; settles facts empirically in a Secure Boot + TPM sandbox; shows no
  Debian-derived shim prevents (Debian's shim embeds the Debian CA), so
  prevention requires an OWN Kicksecure shim (own VENDOR_CERT, via shim-review
  and/or Sovereign own-key) chainloading a Kicksecure-signed UKI (Debian
  kernel+initrd) over dm-verity, MOK preserved; measured boot + TPM-sealed FDE as
  defense-in-depth; IMA measurement-only.
