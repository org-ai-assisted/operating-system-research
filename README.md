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
