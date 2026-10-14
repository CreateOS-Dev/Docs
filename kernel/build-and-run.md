# Build, run, and image safety

## Host prerequisites

The build requires Make, a host C compiler, NASM, x86-64 ELF cross-GCC and
binutils, GRUB image tooling, and xorriso. QEMU is required for VM runs. On
macOS, the disposable USB image helper also uses `hdiutil` and `newfs_msdos`.
`make check-build-tools` checks the build tools available on `PATH`.

Install supported host dependencies with:

```sh
make install-deps
make check-deps
```

The installer supports Homebrew on macOS and apt on Debian/Ubuntu. On macOS it
requires Xcode Command Line Tools and installs NASM, QEMU, xorriso, GRUB
(`i686-elf-grub`), GNU build prerequisites, and download utilities. On
Debian/Ubuntu it installs the corresponding compiler/build libraries,
`qemu-system-x86`, `grub-pc-bin`, NASM, and xorriso. `make check-deps` checks
the full build-and-QEMU tool set without installing packages.

## Cross-toolchain

Build the pinned cross-toolchain locally:

```sh
make toolchain
make verify-toolchain
```

The default prefix is `build/toolchain/x86_64-elf`. Set `TOOLCHAIN_PREFIX`,
`TARGET`, or `JOBS` to override it. Source archives are SHA-256 checked before
use. The verification target checks `gcc -dumpmachine`, compiles a freestanding
C probe, links it with the selected cross-linker, and checks the output symbol.
The kernel and user applications use the selected `CROSS_COMPILE` prefix.
Build targets fail rather than falling back to similarly named tools on
`PATH`. The native host compiler is used only to build the cross-toolchain
itself and host-side utilities.

The build scripts are kept in the repository-root [`scripts/`](../scripts/)
directory. `make toolchain` checks its host build prerequisites, builds the
pinned GNU toolchain, and runs the toolchain verification automatically. If the
verified compiler is already installed, the script exits successfully without
rebuilding it.

Remove the installed cross-toolchain only with an explicit confirmation:

```sh
make delete-toolchain CONFIRM=YES
```

This removes only `build/toolchain/<TARGET>`. To remove the cached GNU sources
and intermediate build trees as well, use the separate command:

```sh
make delete-toolchain-sources CONFIRM=YES
```

Both deletion targets are limited to the repository's `build/toolchain`
locations and require `CONFIRM=YES`.

All kernel and guest userland compilation targets require the local
`$(TOOLCHAIN_PREFIX)/bin/$(TARGET)-*` tools. Passing a host compiler or a
different cross-compiler on the command line does not override that
requirement.

`make toolchain-tcc` builds native TinyCC for the host. That is not a guest
compiler and does not install TCC in the CreateOS rootfs.
The normal userland build uses the pinned TinyCC sources under
`kernel/lib/third_party/tinycc` to build the guest compiler as
`/app/system/bin/tcc.APPL`, with a `/bin/tcc.bin` command wrapper. TinyCC is
not linked into the kernel. The build installs CreateOS headers and the static
guest runtime under `/usr/include` and `/usr/lib/tcc`; those guest files are
refreshed on the persistent disk when building an ISO.

## Build and run

```sh
make -j4
make run
```

`make serial` starts the serial-console image. The repository currently has no
maintained automated test target; see the [roadmap](./TODO.md) for verification
gaps. A successful build verifies compilation and linking only, not runtime
behavior or POSIX conformance.

Run `make menuconfig` to configure the supported boot console, kernel log
verbosity, and GRUB menu timeout. The menu writes the ignored local `.config`;
the next build applies it to the generated ISO. These are boot/build settings,
not kernel driver-selection switches.

## Writing an ISO to removable media

The project no longer includes a USB flasher or Makefile target. Build the ISO
with `make os.iso` and use a separate, trusted disk-imaging utility to write it
to removable media. Writing an image erases the selected device; verify its
identity and use media that does not contain data you need. A USB boot is a
live CreateOS medium, not an installer for the computer's internal disk.

The boot image currently relies on GRUB's Multiboot2 loader and the kernel
targets x86-64 PCs. Booting on physical hardware, UEFI/Secure Boot behavior,
installation to a USB drive, firmware compatibility, and the target laptop
have not been validated. This is not an Android Fastboot image or protocol.
Use disposable removable media and retain the machine's existing recovery
media when attempting hardware tests.

## Opt-in QEMU debugging

Normal `make run` and `make serial` do not enable QEMU's verbose emulator
tracing. Set `QEMU_LOGS=1` or `QEMU_LOGS=2` explicitly to enable the existing
trace options for a normal run. For dedicated debugging sessions, use:

```sh
make debug          # graphical guest; serial output is saved to build/qemu-serial.log
make debug-serial   # serial console in this terminal
make debug-monitor  # QEMU monitor in this terminal; serial output saved to a file
make debug-gdb      # pause at reset and listen for GDB on TCP port 1234
```

These targets enable detailed QEMU tracing in `build/qemu-debug.log`; the
graphical, monitor, and GDB targets also save guest serial output to
`build/qemu-serial.log`. `debug-serial` displays that output directly.
`make debug-gdb` waits at reset and opens QEMU's GDB remote stub on TCP port
1234. Connect from another terminal using a GDB-remote-compatible debugger
with symbols from `build/CreateOS` (for example, LLDB's `gdb-remote
localhost:1234` command). The debug targets boot from the requested ISO but use a
temporary copy of `build/disk.img`, removed when QEMU exits, so guest disk writes
from a debug run do not alter the persistent image. The generated trace and
serial logs remain under `build/` for inspection and are removed by `make
clean`. These options are host-side development tools: they are not compiled
into or enabled by the normal guest image.

## Persistent image precautions

`build/disk.img` is persistent guest data. `make run` uses it but does not recreate
an existing image. `make edit` explicitly rebuilds it from staged rootfs
contents. Back up important data before using `make edit`. `make clean` removes
generated kernel, userland, rootfs staging, test, and ISO outputs while
preserving the installed toolchain and its cached sources under
`build/toolchain/`. It does not remove `build/disk.img`. The main boot ISO is
written to `os.iso` in the repository root. `make compressed-release` also
creates `CreateOS-kernel.elf.xz` and `CreateOS-userland.img.xz` there. Use
disposable copies for
filesystem and storage experiments; do not attach a host disk containing
valuable data to the guest.

Generated kernel objects, rootfs staging, ISO staging, and test outputs are
under `build/`.
