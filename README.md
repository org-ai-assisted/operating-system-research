# operating-system-research

Research notes on operating-system design questions: how comparable projects
solve a problem, what is idiomatic, and which design we chose and why.

Each document states the problem, surveys how real projects (named, with
citations) solve it, judges the options, and records the decision.

## Index

- [dasharo-virtualbox-port.md](dasharo-virtualbox-port.md) -- can real Dasharo
  (coreboot + `DasharoPayloadPkg`) firmware run inside VirtualBox? Finds that
  Dasharo is a coreboot distribution (not an OVMF variant), that VirtualBox's EFI
  slot parses only OVMF-shaped flash and rejects a flat `coreboot.rom`, and records
  the minimal ring-3 `DevEFI` loader change that makes coreboot boot and drive
  VirtualBox's chipset (proven), plus the clean next step toward a first-class
  `emulation/qemu-vbox` coreboot board. Code: `org-ai-assisted/virtualbox` and
  `org-ai-assisted/dasharo`.
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
- [nat-vs-bridged-vm-networking.md](nat-vs-bridged-vm-networking.md) -- which VM
  networking mode for a Whonix-Gateway. Separates the two families (user-mode
  connection-terminating NAT -- VirtualBox NAT, KVM passt/SLIRP -- versus on-LAN
  L2/L3 bridged/macvtap), with VirtualBox/libvirt/passt citations. Records the
  empirical finding that user-mode NAT regenerates the TCP ISN (so in-VM `tirdad`
  is bypassed; verified in network namespaces AND in a full Whonix-Gateway VM),
  while L3 forwarding (a home router's NAPT, Qubes, bare metal) preserves it.
  Fact-checks Whonix's "bridged is bad" stance against the `whonix-gateway-firewall`
  source and finds it mainly historical/"untested" caution -- the anonymity model is
  link-independent -- with one substantiated additive concern (local-LAN / MAC
  exposure). Decision: keep NAT/passt, fix the ISN with host-side `tirdad`. Code:
  `org-ai-assisted/dist-ai` `usr/share/tirdad-isn-nat-tests/`.
