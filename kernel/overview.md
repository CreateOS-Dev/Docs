# System overview

## Boot path

The BIOS/GRUB boot path loads a Multiboot2 x86-64 kernel. The kernel sets up
architecture state, interrupts, memory management, the console and timer, then
enumerates PCI devices and initializes available drivers. It initializes the
VFS, mounts the boot-provided CFS5 image, and starts the init/login/shell
userland.

The exact initialization order lives in `kernel/main.c`. Boot messages are
written to the serial console and framebuffer/VGA output where those devices
are available. Use the serial QEMU target when graphical output is not useful.

## Layout

- `boot/`: boot assembly and GRUB configuration.
- `kernel/arch/`: architecture-specific low-level code.
- `kernel/header/`: kernel interfaces, separated from implementation files.
- `kernel/drivers/`: console, input, platform, network, storage, USB, audio,
  and clock drivers.
- `kernel/memory/`, `kernel/proc/`, `kernel/security/`: memory, user execution,
  and security-related code.
- `kernel/fs/`: VFS and filesystem implementations.
- `rootfs_maker/lib/`: guest syscall wrappers and the small POSIX-facing
  compatibility library.
- `rootfs_maker/root/`: userland source, assets, and rootfs helper scripts.
- `rootfs_maker/tools/`: host-side image creation and inspection utilities.
- `build/`: generated build output and staging; not source of record.

## Current scope

The checked-in kernel build is for x86-64. The `i686-elf` GRUB utility, where
used, is only for producing a BIOS boot image; it is not an i386 kernel target.
The system is a development OS, and successful compilation or QEMU startup
does not imply complete hardware, POSIX, networking, or security support.
