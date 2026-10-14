# Kernel notes

The kernel is freestanding C with x86-64 assembly for the parts that need to
touch CPU state directly. The public interfaces are mostly collected under
`kernel/header/`; implementations sit beside them in `kernel/`.

## Startup and CPU state

`boot/boot.S` is the Multiboot2 entry point. The kernel's architecture code
sets up descriptor tables, interrupt handling, the syscall entry path, task
state, and the initial kernel stack. Start with `kernel/main.c` to follow the
initialization sequence. Boot messages go to the active TTY and serial output.

## Memory and user programs

`kernel/memory/` contains the physical-frame allocator and virtual-memory
mapping code. User programs are ELF64 images loaded into a separate user
address space. Before the kernel copies syscall arguments or buffers, it
checks that the supplied user addresses are readable or writable as needed.

`kernel/proc/` handles executable loading, account state, and synchronous
process operations. `exec` replaces the current user image; `sys_run` can run
a child image and return its status. The `fork`/`wait` subset is serialized:
it is not a preemptive, concurrently scheduled process model.

## System calls

The syscall numbers and user-facing declarations are in
`kernel/header/syscall.h` and `rootfs_maker/lib/user_syscall.h`. Entry and
register handling are in `kernel/arch/x86_64/syscall.asm`; handlers are in
`kernel/syscall.c`.

When changing the ABI, update both kernel and userland declarations. Check
argument count, pointer validation, path permissions, and the behavior on a
failed copy. See [the syscall guide](../userland/syscalls.md) for the current
groups and conventions.

## Filesystems

The VFS connects the kernel to filesystem drivers. CFS5 is the root
filesystem and is implemented in `kernel/fs/`; its host tools live in
`rootfs_maker/tools/`. The disk image is built separately from the kernel ELF
and included in the GRUB ISO as a boot file.

## A useful path through the code

For a small kernel change, trace the feature from its header to its caller and
then to its implementation. For example, a filesystem syscall passes through
the user wrapper, syscall entry, handler and permission check, then the VFS
and CFS5 driver. Keeping that path intact is usually more important than
adding another helper layer.

Build and run changes using [the build guide](./build-and-run.md). Kernel
compilation alone does not exercise interrupt timing, hardware behavior, or
user/kernel transitions; use QEMU for changes that affect runtime.
