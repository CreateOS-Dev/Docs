# Build and run

The build happens on macOS or Linux. CreateOS uses a pinned x86-64 ELF
cross-toolchain for the kernel and guest programs; the compiler installed on
your host is not used as a fallback.

## Set up the host

Install the supported build dependencies:

```sh
make check-deps
make install-deps
```

`make check-deps` reports missing packages without installing anything.
`make install-deps` supports Homebrew on macOS and apt on Debian/Ubuntu. You
will also need the Xcode Command Line Tools on macOS.

Build and verify the cross-toolchain:

```sh
make toolchain
make verify-toolchain
```

The default toolchain is placed in `build/toolchain/x86_64-elf/`. The verify
target checks the compiler target and compiles and links a small probe.

## Build the ISO

```sh
make
make run
```

`make` builds the userland, updates the disk image, links the kernel, and
creates `os.iso`. `make run` starts it in QEMU. For a terminal-only session:

```sh
make serial
```

The first boot opens account and regional setup. Later boots go straight to
login. `make menuconfig` changes the supported boot console, log level, and
GRUB timeout in the local `.config`; it does not choose which drivers are
compiled.

Useful targets:

| Command | Purpose |
| --- | --- |
| `make` | Build the normal ISO |
| `make run` | Run the ISO and disk in QEMU |
| `make serial` | Run with the guest serial console in the terminal |
| `make debug-serial` | Start a serial debug run with QEMU tracing |
| `make disk-image` | Rebuild the CFS5 disk image from staged userland |
| `make refresh-system-apps` | Refresh staged apps and update the existing disk |
| `make compressed-release` | Build the ISO and a compressed distribution archive |

The debug run targets use a temporary copy of the disk image. Normal `make
run` uses `build/disk.img`.

## Keep your disk image

`build/disk.img` can contain accounts and files you have created. Back it up
before rebuilding or cleaning. `make clean` removes the disk image along with
other generated outputs. `make edit` also rebuilds the image, then rebuilds
the ISO. `make run` does not intentionally reformat an existing disk image.

To start over, first save anything you need, then run:

```sh
make disk-image
make
```

## Hardware and removable media

The normal development path is QEMU. Building an ISO does not make it a tested
installer. There is no supported USB flashing target in this repository.
If you write `os.iso` to removable media with another utility, double-check
the selected device: imaging erases its contents. Physical boot and
firmware-specific behavior still need separate testing.

## Compressed output

`make compressed-release` writes `build/CreateOS-x86_64-compressed.tar`.
It contains compressed kernel and rootfs inputs plus GRUB configuration. The
current boot path does not decompress these at boot; unpack and decompress
them before using the included configuration.
