# System-call ABI

The user ABI declarations and numeric constants are maintained in
`rootfs_maker/lib/user_syscall.h` and the corresponding kernel syscall
implementation. Keep both sides synchronized when changing a syscall.

The current interface includes console/input and cursor operations, file and
directory operations, process-related calls, user/account operations, memory
helpers, TTY controls, mount/sync/fsck functions, USB information, networking,
history, and kernel-version queries. The definitive syscall numbers and C
signatures are in the header; this document intentionally avoids duplicating
that list and becoming stale.

## Error and compatibility behavior

Raw `sys_*` wrappers generally return a negative error value on failure.
POSIX-facing wrappers in `rootfs_maker/lib/posix.c` translate those failures
to `-1` and set the global `errno`. Verify each implementation for its actual
error mapping; this is not a complete POSIX ABI.

There is currently no dedicated automated POSIX compile/link or behavior test
target. A successful system build does not establish syscall behavior,
standards conformance, ABI compatibility with Linux, or application
portability. New ABI calls need argument validation, user-memory access checks,
kernel implementation, a user wrapper, and focused positive/negative tests.
