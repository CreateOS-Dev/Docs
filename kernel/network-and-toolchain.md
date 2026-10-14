# Networking and toolchain status

## Networking

CreateOS network devices register through the kernel `netdev` interface.
lwIP's Ethernet input owns all received Ethernet frames; QEMU's emulated E1000
and user-mode networking are the primary development setup. This is not yet a
general-purpose Internet or POSIX sockets implementation.

lwIP 2.2.0 sources and their upstream license are vendored under
`kernel/drivers/network/lwip/`. Selected core, IPv4, and Ethernet modules are
compiled into the kernel with `NO_SYS=1`; the active Ethernet RX, ARP,
IPv4/IPv6, DHCPv4, DNS, ICMP, and TCP paths are owned by lwIP. CreateOS retains
the `netdev` hardware interface and compatibility syscalls/commands, backed by
lwIP's DHCP/DNS/raw-ICMP/raw-TCP APIs. The former custom IPv6 protocol engine
and hand-built IPv4/DHCP/DNS/ICMP/TCP packet handlers have been removed.
`curl` now has a user-space TLS client based on Mbed TLS 4.0.0: HTTPS permits
TLS 1.2 through TLS 1.3, requires a trusted certificate chain and matching
hostname, and never falls back to plaintext. Its TLS transport uses the
kernel's persistent TCP stream syscalls; hardware entropy is mandatory and
HTTPS fails closed when the x86 RNG or RTC is unavailable. The RTC is
interpreted as UTC with years 2000-2099; clocks earlier than 2020 and invalid
calendar values are rejected. The current URL/network interface is IPv4-only.
TCP stream handles are positive signed 32-bit values, bound to the opening
console and user-image generation, and invalidated during user-image
transitions. Closing a live stream as part of an image transition still needs
an end-to-end QEMU test.
System-privileged applications can also open one IPv4 TCP listener on a
selected interface and port, accept one connection at a time, and use the
existing stream read/write/close calls. Closing a listener also closes its
currently accepted stream. The public app SDK exposes it as
`<createos/network.h>` and `libcreateos-network.a`. This custom API is not
POSIX sockets and does not provide an ingress firewall. The built-in
`httpd.APPL` uses the API for a single-request diagnostic endpoint on TCP port
8080; a QEMU user-network port-forward smoke test confirmed `GET /health`
receives HTTP 200. Listener-handle ownership, cleanup during process
transitions, and denial of unowned handles still need QEMU validation.
The root CA bundle is Mozilla-derived, dated 2026-09-25, and installed at
`/etc/ssl/cert.pem` (SHA-256
`a41b5d356aea97a529fe27e0f7316d2f9d946d75927476cf9cf1b90637d00505`); refresh
it from the upstream curl CA extract and review the bundle hash before
updating. General-purpose HTTP support, POSIX sockets, DHCPv6, and a
route-management API remain separate/incomplete features. HTTPS has been
compiled and linked, but QEMU success and negative-certificate tests are still
required before claiming runtime validation. QEMU user-mode tests with the
E1000 have verified a DHCPv4 lease
(`10.0.2.15/24`, gateway `10.0.2.2`, DNS `10.0.2.3`), DNS resolution and an
HTTP response from `example.com`, an ICMP echo reply from `1.1.1.1`, and a
guest-to-host TCP HTTP request/response. The legacy and modern VirtIO-net
transports have both passed QEMU user-mode tests for DHCPv4, ICMP echo from
`1.1.1.1`, and DNS plus HTTP from `example.com` (`HTTP/1.1 200 OK`). These
runs used disposable root filesystem images; the modern-only test used
`virtio-net-pci,disable-legacy=on` and captured the DHCP, ICMP, DNS, and TCP
HTTP exchanges. Its serial transcript and packet capture are saved as
`build/virtio-modern-icmp-dns-http.log` and
`build/virtio-modern-icmp-dns-http.pcap`. These are smoke tests, not a complete
protocol conformance or malformed-packet test suite. The E1000 tests also
exposed and fixed syscall argument marshalling for four-or-more arguments and
the IPv4-header offset used by lwIP raw ICMP callbacks.

IOAPIC timer routing is not enabled at runtime until periodic ticks and the
same network tests pass with that interrupt path.

The current drivers include Intel E1000, legacy Realtek RTL8139, and legacy
and modern PCI VirtIO-net transports. The VirtIO driver uses the shared
`netdev` interface and polled virtqueues. DHCP, ICMP, DNS, and HTTP have been
smoke-tested with both VirtIO transports. Realtek RTL8111 is not implemented.

For a serial QEMU boot using the legacy VirtIO transport, override the default
E1000 arguments:

```sh
make debug-serial \
  QEMU_NET_ARGS='-netdev user,id=createos-net,restrict=off -device virtio-net-pci,disable-modern=on,netdev=createos-net'
```

For a modern-only VirtIO test, use `disable-legacy=on` instead:

```sh
make debug-serial \
  QEMU_NET_ARGS='-netdev user,id=createos-net,restrict=off -device virtio-net-pci,disable-legacy=on,netdev=createos-net'
```

DHCP autostart is enabled by default through the startup hook used by init and
first-boot setup. Init retries DHCP discovery once if the first five-second
wait expires, then continues to login even if no lease is available. The staged
root filesystem contains `/etc/network.conf`:

```ini
autostart=1
```

Set it to `autostart=0` to disable DHCP at startup; `net dhcp 0` remains
available for a manual request. The current guest shell has no script
interpreter, so this hook is a compiled userland startup routine rather than a
shell script.

The network driver sources are not complete FreeBSD ports. A driver must be
adapted to CreateOS interfaces, reviewed for licensing and correctness, and
tested on the target hardware before compatibility is claimed.

## Toolchain boundaries

- `make toolchain`: pinned GNU x86-64 cross-toolchain under `build/`.
- `make verify-toolchain`: target check plus compile/link/symbol smoke test.
- The repository-root `Makefile` builds the kernel, user programs, and host
  bootable ISO; it is run on macOS/Linux, not by the guest OS.
- NASM, cross GCC/binutils, and `grub-mkrescue` are build-host tools. GRUB is
  included as the ISO bootloader; its host image-creation utility is not a
  guest compiler or a command installed in CreateOS.
- `make toolchain-tcc`: pinned TinyCC built to run on the host.

Host TinyCC is not a guest compiler and is not installed in the rootfs.
On macOS Apple Silicon, the pinned 0.9.27 build uses TinyCC's x86-64 Darwin
code generator while the compiler executable itself remains a host binary.
The regular rootfs build separately compiles pinned TinyCC from
`kernel/lib/third_party/tinycc` for CreateOS and installs it as a ring-3
`.APPL`, with CreateOS headers and static guest runtime archives in the
rootfs. Kernel services remain behind the existing syscall interface;
compiler implementation and guest runtime code are not linked into the
kernel.
The guest shell also lacks script execution and general process spawning, so
copying host NASM, GNU make, or GRUB binaries into the root filesystem would
not make them executable. Guest NASM and build orchestration require native
guest ports and tested runtime interfaces; producing a bootable ISO from
inside CreateOS additionally requires a supported disk/ISO writer.
The current build exposes the pinned x86-64 GCC/binutils cross-toolchain and
host TinyCC as separate host tools, as well as a separately built guest TinyCC.

## Compressed release artifacts

`make compressed-release` first builds the ordinary bootable ISO, then writes
`build/CreateOS-x86_64-compressed.tar`. The archive contains an XZ-compressed
kernel ELF, an XZ-compressed CFS5 root filesystem disk image, and the GRUB
configuration. The source `CreateOS` kernel and `build/disk.img` are read-only inputs
to this packaging target.

This is a distribution bundle, not yet a directly bootable compressed image:
the current Multiboot2 entry still loads an uncompressed ELF kernel and disk
image. Extract and decompress both `.xz` files before using the included GRUB
configuration. Boot-time XZ decompression and mounting the compressed root
filesystem are not implemented.
