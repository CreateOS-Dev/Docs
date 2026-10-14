# Networking and build tools

This page covers two things that are easy to confuse: network programs that
run inside CreateOS, and compilers/tools that run on the development machine.

## Network stack

Network devices register through the kernel `netdev` interface. The current
stack uses lwIP in a polling configuration (`NO_SYS=1`); the source and its
license are under `kernel/drivers/network/lwip/`. Drivers in the tree include
Intel E1000, RTL8139, and legacy/modern VirtIO-net paths. QEMU's E1000 with
user networking is the default development setup.

The guest has DHCPv4, DNS, ICMP echo, and outbound TCP support. The `net`,
`ping`, and `curl` commands use these facilities. `curl` supports HTTPS using
Mbed TLS, a trusted CA bundle, a valid guest clock, and hardware entropy.
Certificate or entropy failures should stop an HTTPS request rather than
silently downgrade it.

System-privileged applications can use a limited TCP listen/accept API:

- one listener at a time;
- one accepted stream at a time;
- custom CreateOS calls, not POSIX sockets;
- no general inbound firewall.

The `httpd.APPL` application is a one-request diagnostic server on port 8080,
not a production web server. QEMU user networking needs an explicit port
forward before a host can connect to it.

The guest shell does not execute shell scripts, and networking features are
not a claim of full protocol conformance. Test the exact driver and protocol
path you intend to use.

## Cross-toolchain

The root Makefile builds the x86-64 kernel and userland with a pinned
`x86_64-elf` GNU toolchain under `build/toolchain/`. Check or install the
host dependencies and build it with:

```sh
make check-deps
make install-deps
make toolchain
make verify-toolchain
```

The `x86_64-elf` compiler creates guest executables; the host compiler is used
for tools that run on macOS or Linux, such as the CFS5 image utilities.

There are two TinyCC builds:

- `make toolchain-tcc` builds a host TinyCC used by build tooling.
- The userland build compiles the pinned TinyCC source into
  `/app/system/bin/tcc.APPL`, which runs inside CreateOS.

The guest compiler builds static CreateOS executables. It does not add dynamic
linking or make host tools such as `make`, NASM, or GRUB available in the
guest.

## Compressed release

`make compressed-release` creates `build/CreateOS-x86_64-compressed.tar`,
including an XZ kernel and userland image. The ordinary GRUB boot path still
expects an uncompressed kernel and disk image. Decompress the archive contents
before using them; boot-time decompression is not implemented.
