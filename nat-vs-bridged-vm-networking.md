# VM networking: NAT versus bridged (and the other modes)

## Problem

A virtual machine reaches the outside world through one of a few networking modes,
and the choice is usually made by inertia (the virtualizer's default) rather than by
argument. For an anonymity gateway (Whonix-Gateway) the choice has real security and
reliability consequences, and it was thrown into relief by a concrete finding: the
default user-mode NAT silently discards the guest's TCP initial sequence number
(ISN), which defeats `tirdad` (the Kicksecure TCP-ISN CPU-information-leak
protection) when it runs inside the VM.

This note records how each networking mode actually works, what the ISN finding is
and why it happens, how the modes compare on the axes that matter (ISN/`tirdad`,
local-LAN exposure, leak-proofness, reliability, configuration complexity), and
whether Whonix's long-standing "NAT good, bridged bad" position is substantiated or
mainly historical. The ISN reproduction is encoded as a regression suite in
`org-ai-assisted/dist-ai` (`usr/share/tirdad-isn-nat-tests/`, opt-in e2e lane).

## The two families (the pivotal distinction)

Every mode is one of two kinds, and almost everything below follows from which kind
it is:

- **User-mode / connection-terminating** -- the host runs a userspace TCP/IP stack
  that TERMINATES the guest's TCP connection and opens a fresh host socket to the
  real destination. The guest is invisible on the physical network; egress looks
  like it came from the host. Self-contained: no host-network cooperation.
  - VirtualBox **NAT** -- "acts much like a real computer that connects to the
    Internet through a router", with the VirtualBox engine as that router and a
    built-in DHCP server; the guest "is invisible and unreachable from the outside"
    and traffic "looks like the data was sent by the Oracle VM VirtualBox
    application on the host" (VirtualBox manual, ch.6).
  - VirtualBox **NAT Network** -- the same libslirp-derived stack shared by a group
    of VMs; still not on the physical LAN (inbound needs explicit port forwarding).
  - QEMU/KVM **SLIRP** (`-netdev user`) -- the same libslirp library VirtualBox's
    NAT is derived from; default guest 10.0.2.15, gateway 10.0.2.2.
  - QEMU/KVM **passt/pasta** (the modern libvirt default) -- "a translation layer
    between a Layer-2 network interface and native Layer-4 sockets"; unprivileged,
    creates no host interface, presents the guest as the host's own IP, and is
    namespaced + seccomp-confined (passt.top; libvirt).

- **On-LAN / L2-L3 forwarding** -- the guest's frames reach the physical network
  (near) unchanged; the guest is a first-class host on the LAN with its own MAC and
  IP, served by the LAN's DHCP. Requires host cooperation (a bridge device,
  promiscuous mode).
  - VirtualBox **Bridged** -- a host net-filter driver attaches the guest so that
    "it looks ... as though the guest were physically connected to the interface
    using a network cable" (VirtualBox manual).
  - KVM/libvirt **bridge** / **macvtap** -- "full incoming and outgoing net access
    just like a physical machine"; the IP range "is whatever is used on the LAN";
    needs a host bridge with a physical NIC attached (libvirt domain XML).

A home-router NAPT, for contrast, is a THIRD thing: it rewrites source
address/port at L3 and forwards the same TCP segment. It does NOT terminate the
connection, so it does not regenerate the ISN -- it behaves like the on-LAN
forwarding family for the ISN, not like user-mode NAT.

## The ISN finding (why user-mode NAT defeats guest tirdad)

`tirdad` randomizes the TCP ISN in the kernel that ORIGINATES a connection's socket.
Under user-mode NAT the socket that reaches the wire is the HOST's (libslirp/passt
opened it), so the on-wire ISN is the host kernel's and the guest's `tirdad`-chosen
ISN never leaves the host. Under L2/L3 forwarding the guest's own segment reaches the
wire, so the guest ISN (and `tirdad`) survives.

Verified empirically (2026-10):

- **Namespace reproduction** (faithful stacks: `pasta` IS passt's engine,
  `slirp4netns` IS libslirp). Measuring the guest-side SYN ISN versus the wire-side
  SYN ISN for the same connection:
  - libslirp (== VirtualBox NAT family): ISN **regenerated** (guest != wire).
  - passt (== KVM default): ISN **regenerated**.
  - routed veth + host `ip_forward` + MASQUERADE (== a home router's NAPT, Qubes-Whonix,
    bare metal): ISN **preserved** (guest == wire).
- **Full-VM reproduction** -- a real Whonix-Gateway-CLI 18.2.4.2 OVA under real
  VirtualBox 7.2 NAT, `tirdad` reported Enabled by `systemcheck`, during a real Tor
  bootstrap. Capturing the guest frames (VirtualBox `--nictrace`) versus the host
  wire (`tcpdump`): two Tor guard connections, both with the guest ISN fully replaced
  on the wire (e.g. 3502149445 -> 1705308543; 632236576 -> 1025303812), and the
  source port and source IP rewritten too -- the signature of full TCP termination.

So for both default non-Qubes Whonix setups (VirtualBox NAT and KVM passt), in-VM
`tirdad` does not influence the on-wire ISN. The on-wire ISN is the HOST kernel's.

### Consequence and fix

Because the wire ISN is the host's, the protection that matters is `tirdad` ON THE
HOST (Kicksecure host, or the module loaded on a Linux host) -- then the NAT engine's
own host socket gets a strong ISN. Switching the VM to bridged would also fix it (the
guest ISN then reaches the wire), but at the cost of the on-LAN exposure and fragility
below. On Windows/macOS hosts `tirdad` (a Linux module) cannot run, so neither
host-`tirdad` nor the Linux-specific leak it addresses applies; that is a host-OS
property outside the VM's control. In-VM `tirdad` stays harmless and IS effective
where the path forwards at L3 (Qubes-Whonix, bare-metal Kicksecure).

## Comparison by axis

| Axis | User-mode NAT (VBox NAT / passt) | On-LAN bridged (VBox bridged / KVM bridge, macvtap) |
| --- | --- | --- |
| On-wire TCP ISN / `tirdad` | host regenerates it; in-VM `tirdad` bypassed (fix: host `tirdad`) | guest ISN preserved; in-VM `tirdad` effective |
| Local-LAN exposure / isolation | guest invisible + unreachable from the LAN; egress rebranded as the host | guest is an L2 citizen of the LAN -- own MAC+IP, reachable from and able to reach the router + other LAN hosts |
| Leak containment (defense in depth) | a stray or uncovered packet is contained to the private NAT net; the gateway identity never touches the LAN | no containment: stray L2/IPv6-RA/NDP/DHCP/mDNS and the gateway MAC reach the real LAN |
| Anonymity of traffic (exit IP) | Tor exit (firewall-enforced) | Tor exit -- SAME; the firewall is link-independent |
| Configuration / reliability | self-contained; built-in DHCP; no host cooperation; works on Wi-Fi, captive, port-security nets | needs a host bridge + promiscuous mode, the LAN's DHCP, the host firewall not blocking it; Wi-Fi: bridged generally FAILS (802.11 single-MAC, see below); breaks on port-security / captive portals |
| Whonix support | default, supported | "unrecommended / untested / breaks networking" |

Notes on the two subtle rows:

- **Anonymity is equal.** Whonix's leak protection is the Gateway firewall, and it is
  pure L3/L4: default `policy drop` on input, `reject` on forward and output
  (fail-closed), and a NAT redirect of all Gateway clearnet TCP into Tor's TransPort
  except whitelisted UIDs (`debian-tor` etc.). Nothing in it references the external
  adapter's nature, so whether the uplink is a NAT adapter or a bridged adapter,
  clearnet is still forced through Tor and the box still fails closed. Bridged does
  NOT break the Tor-forcing or fail-closed guarantee (verified against the
  `whonix-gateway-firewall` source).
- **Leak containment differs.** What bridged ADDS is local: the Gateway gets a real
  LAN IP+MAC (the router "may see the gateway MAC address which identifies as
  Whonix-Gateway"), and the firewall's hardcoded clear-net LAN whitelist
  (192.168.0.0/24, 192.168.1.0/24, 10.0.2.2/24) is a live surface on a real LAN
  rather than the tiny isolated 10.0.2.x of NAT. That is additive exposure, not a
  collapse of anonymity.

## Wi-Fi: bridged usually fails outright

On a Wi-Fi uplink, bridged is not merely "fragile" -- it typically does not work at
all, which matters because laptops are the common Whonix host. The root cause is
802.11 itself: in infrastructure mode a station associates with the access point
using ONE MAC address, and the AP then accepts frames only from that associated MAC
and discards frames carrying any other source address. A bridge forwards the guest's
frames unchanged, so the guest's own MAC reaches the AP and is dropped and the guest
gets no connectivity. This is not virtualizer-specific: Proxmox's WLAN guidance states
it plainly and recommends against bridging over WLAN, and FreeBSD/bhyve users hit the
same wall. A guest can even obtain a DHCP lease yet still be dropped, because some APs
additionally bind the leased IP to the associated MAC (Ruckus).

VirtualBox papers over it only partially, by rewriting MAC addresses, because "most
wireless adapters do not support promiscuous mode": it "needs to replace the source
MAC address in the Ethernet header of an outgoing packet" with the host Wi-Fi
adapter's MAC, replaces the destination MAC on the reply, and "examines ARP and DHCP
packets in order to learn the IP addresses of virtual machines" to demultiplex those
replies (VirtualBox manual). Because the demux depends on tracking L3 addresses, this
works for IPv4 and IPv6 ONLY -- "for other protocols, such as IPX, you must choose a
wired interface" -- on macOS (AirPort) and Linux; Oracle Solaris has "no support for
using wireless interfaces" at all.

KVM/libvirt bridge and macvtap have no such built-in rewrite, so bridging a guest over
a Wi-Fi station interface generally does not work there either. The documented
options are a MAC-rewriting ebtables hack, 4-address/WDS mode (only if the AP supports
it), or -- the standard recommendation -- NAT.

So on Wi-Fi the practical choice for a VM gateway is user-mode NAT; bridged is a
wired-LAN option. This is a concrete, substantiated reliability reason for the NAT
default, independent of the security arguments above.

## Is Whonix's "bridged is bad" substantiated, or historical?

Mostly historical/caution, with one substantiated-but-additive concern.

- **Not a demonstrated security requirement.** The wiki's entire stated rationale is
  "Do not change ... to a bridged network! This is unsupported, untested and should
  not be necessary." Whonix's own definition of that label (the `Unsupported` page)
  is explicitly non-security: "undocumented, untested, or unsupported ... does not
  necessarily imply technically impossible". "Breaks networking" in the
  troubleshooting page traces to OPERATIONAL reports (relay failures after the
  switch, Wi-Fi bridging unavailable), not a leak demonstration.
- **No rejection on merit is on record.** The one reachable origin ticket
  (Phabricator T49 / forum topic 19348) is Patrick Schleizer proposing to first WRITE
  a NAT-vs-bridged comparison chapter and THEN decide the default -- a chapter never
  produced. NAT reads as the inherited VirtualBox default that worked. The two
  original SourceForge debate threads the wiki cites are gone (404; the Wayback
  snapshots captured only SourceForge's error page), so any sharper argument they
  held is not citable today.
- **The firewall analysis clinches link-independence** (above): the anonymity model
  does not depend on NAT.
- **The tell that it is not fundamental:** Whonix's own Physical Isolation guidance
  runs the Gateway on SEPARATE HARDWARE using its physical NIC directly on the real
  LAN (DHCP from the router, real MAC) -- topologically identical to bridged -- and
  treats it as a supported, valid architecture. The same link exposure is endorsed
  when it is the intended design; the only caveat raised there is MAC exposure.
- **The one substantiated reason** to prefer NAT for the default single-host case is
  therefore the additive local-LAN / MAC exposure plus the larger clear-net LAN
  whitelist surface -- real and defensible, but it widens exposure rather than
  breaking anonymity.

## Decision

For the default single-host Whonix deployment, keep user-mode NAT (VirtualBox NAT /
KVM passt):

- It gives real isolation (the Gateway is not an L2 citizen of the LAN) and leak
  containment (defense in depth), and it is self-contained and robust across Wi-Fi,
  captive portals, and port-security networks.
- It does NOT give stronger anonymity than bridged -- the firewall provides that
  regardless of link type -- so the honest reason to prefer it is "avoid additive
  local-LAN exposure and keep networking robust," not "NAT is required for
  anonymity."

Fix the ISN regression at the host, not by switching the link:

- Run `tirdad` on the host (Kicksecure host, or load the module on a Linux host), so
  the NAT engine's host socket -- which is what reaches the wire -- gets a strong ISN.
- Keep in-VM `tirdad` (harmless under NAT, effective under L3 forwarding such as
  Qubes-Whonix and bare metal).
- Correct the Whonix wiki: in-VM `tirdad` does not affect the on-wire ISN under NAT;
  host `tirdad` does. Keep the module; do not call it useless.

Bridged remains a power-user exception (e.g. a VM gateway that intentionally serves a
physical LAN), not the default and not the ISN fix.

## Sources

- VirtualBox manual, Virtual Networking (bridged, and the wireless MAC-rewrite + IPv4/IPv6-only limits) -- https://www.virtualbox.org/manual/ch06.html
- libvirt, Domain XML (interfaces: user/passt/bridge/direct) -- https://libvirt.org/formatdomain.html
- Proxmox WLAN wiki (why bridging over Wi-Fi fails; recommends against it) -- https://pve.proxmox.com/wiki/WLAN
- FreeBSD bhyve: no guest network when bridged from a Wi-Fi interface -- https://forums.freebsd.org/threads/no-network-on-guest-when-bridged-from-wi-fi-interface.89148/
- Ruckus KB: AP binds leased IP to the associated MAC (VM gets an IP but is dropped) -- https://support.ruckuswireless.com/answers/000004024
- passt -- https://passt.top/passt/about/
- Whonix-Gateway Security (the bridged warning) -- https://www.whonix.org/wiki/Whonix-Gateway_Security
- Whonix Troubleshooting (virtualizer settings / "breaks networking") -- https://www.whonix.org/wiki/Troubleshooting
- Whonix Unsupported (definition of the label) -- https://www.whonix.org/wiki/Unsupported
- Whonix Physical Isolation (real-NIC gateway topology; MAC exposure) -- https://www.whonix.org/wiki/Physical_Isolation
- Whonix IPv6 (LAN RA exposure) -- https://www.whonix.org/wiki/IPv6
- Whonix-Gateway Firewall -- https://www.whonix.org/wiki/Whonix-Gateway_Firewall
- whonix-firewall source (L3/UID/port, fail-closed, Tor redirect) -- https://github.com/Whonix/whonix-firewall
- "NAT vs Bridged networking for Whonix-Gateway" (origin ticket T49 / forum 19348) -- https://forums.whonix.org/t/nat-vs-bridged-networking-for-whonix-gateway/19348
- ISN reproduction + regression suite -- `org-ai-assisted/dist-ai`, `usr/share/tirdad-isn-nat-tests/`
