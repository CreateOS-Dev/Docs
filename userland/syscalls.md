# System-call interface

User programs enter the kernel through the x86-64 syscall path. The kernel
numbers and handler registration live in `kernel/header/syscall.h` and
`kernel/syscall.c`; guest declarations and wrappers live in
`rootfs_maker/lib/user_syscall.h` and `rootfs_maker/lib/syscalls.S`.

The interface is CreateOS's own ABI. A few names resemble Linux or POSIX, but
matching names do not guarantee identical behavior.

## Call groups

| Area | Examples |
| --- | --- |
| Console and TTY | `SYS_PUTS`, `SYS_GETC`, `SYS_TTY_INFO`, cursor, mouse, clipboard |
| Files and directories | `SYS_OPEN`, `SYS_READ`, `SYS_WRITE`, `SYS_STAT`, `SYS_MKDIR`, `SYS_UNLINK`, `SYS_RENAME`, `SYS_GETDENTS64` |
| Processes | `SYS_EXEC`, `SYS_RUN`, `SYS_FORK`, `SYS_WAIT`, `SYS_GETPID`, `SYS_EXIT` |
| Users | login, logout, account lookup, password verification, roles, lock/enable |
| Memory and time | `SYS_ALLOC`, `SYS_FREE`, `SYS_TICKS`, `SYS_SLEEP`, `SYS_TIME`, monotonic time |
| Filesystem administration | permissions, ownership, mount, sync, `SYS_STATFS`, `SYS_FSCK` |
| Network | interface info, DHCP, ping, DNS, TCP streams, TCP listener calls |
| Devices and system | PCI and USB info, fonts, power settings, random bytes, kernel version |

The complete current number list is the header; this page groups the calls
rather than copying a list that can drift out of date.

## Calling convention and wrappers

Guest C programs should normally call the functions declared in
`user_syscall.h`, rather than issuing the `syscall` instruction directly.
The assembly wrappers place the syscall number and up to six arguments in
the registers expected by the kernel entry code.

The POSIX compatibility layer in `rootfs_maker/lib/posix.c` adapts a small
subset of file, process, and permission calls. It translates errors to
`errno` where appropriate, but it is not a complete libc.

## Pointers, paths, and permissions

Syscall handlers validate user buffers before reading or writing them. A
pointer that is outside the active user mapping, has the wrong access
permission, or describes an invalid string must fail rather than be trusted.

Filesystem calls also resolve paths against the current working directory
and the active account's access rules. Some paths are public to regular
users; others require ownership or administrator privileges. Administrative
network listener calls are restricted to system-privileged programs.

When adding a syscall:

1. Allocate a number in `kernel/header/syscall.h`.
2. Add its user declaration and wrapper as needed.
3. Register the handler with the correct argument count.
4. Validate every user pointer, length, path, and permission.
5. Test both valid input and failure cases from a user program.

Do not assume a positive return value or error convention from another OS.
Follow the existing wrapper for the specific call and check the implementation.

## Process calls

`exec` replaces the current image. `sys_run` runs a child synchronously and
returns its exit status. The current `fork`/`wait` implementation saves a
child image and resumes it when the parent waits; it does not run the child
concurrently and there is no general preemptive scheduler.

See `kernel/syscall.c` and `kernel/proc/appl.c` when changing these paths.
Transitions must revoke or refresh user memory and handles consistently.
