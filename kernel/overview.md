# A tour of CreateOS

CreateOS is built in layers. GRUB loads the x86-64 kernel, the kernel brings
up hardware and memory, mounts CFS5, and starts `/sbin/init`. From there the
userland handles first-time setup, networking, login, and the shell.

## From power-on to the shell

1. GRUB reads the menu under `boot/grub/` and loads the Multiboot2 kernel.
2. The entry code establishes the CPU state needed by the C kernel.
3. `kernel/main.c` initializes the console, CPU and interrupt state, memory,
   timers, device discovery, filesystem, and system services.
4. The VFS mounts the CFS5 disk image supplied with the boot media.
5. The kernel loads `/sbin/init` as the first user program.
6. Init runs first-boot setup when needed, starts configured network
   discovery, then starts `/bin/login`.

The serial log is often the best way to understand a failed boot:

```sh
make serial
```

Use `make debug-serial` when the ordinary log does not show enough detail.

## Where to look

| Path | What lives there |
| --- | --- |
| `boot/` | Multiboot2 entry code and GRUB configuration |
| `kernel/arch/` | CPU setup, interrupts, syscall entry, and context code |
| `kernel/drivers/` | Console, input, clock, PCI, USB, storage, and network drivers |
| `kernel/fs/` | VFS and CFS5 implementation |
| `kernel/memory/` | Physical and virtual memory management |
| `kernel/proc/` | User programs, accounts, and process state |
| `kernel/security/` | User-memory validation and related checks |
| `rootfs_maker/root/` | Guest commands, applications, configuration, and assets |
| `rootfs_maker/lib/` | Guest syscall wrappers and small C/POSIX compatibility layer |
| `rootfs_maker/tools/` | Host-side filesystem creation and inspection tools |
| `sdk/` | Application headers, libraries, rules, and examples |
| `docs/` | Project and development guides |
| `build/` | Generated kernel, disk image, and staging files |

The boot target is x86-64. An i686-prefixed GRUB image tool, where used, makes
a BIOS boot image; it does not mean the kernel is 32-bit.

## What to expect

This is an actively developed OS, mostly exercised in QEMU. Some familiar
ideas are present—users, files, a shell, networking—but compatibility is
selective. Read the [userland guide](../userland/userland.md) and
[network notes](./network-and-toolchain.md) before relying on a particular
command or API.
