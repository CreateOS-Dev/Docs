# Security and trust

CreateOS is experimental software. Treat it as a development system, not as a
place to keep sensitive data or run services on an untrusted network.

## User programs

Applications run in ring 3 with a separate user address space. Syscall
handlers validate user pointers before using them and check file permissions
against the active account. The kernel also tracks system privileges for
selected applications and gates administrative operations.

These boundaries are useful, but they are not a security certification.
Memory management, process transitions, permission changes, and filesystem
recovery continue to evolve. Do not assume that an `.APPL` file is sandboxed:
it is a native executable running with the caller's guest privileges.

## Accounts and files

First boot creates the initial account. Account records live in the root
filesystem; home-directory ownership is assigned from account data. Back up
the CFS5 image before experimenting with account files or filesystem tools.

The `sudo` command and system-privileged applications are part of the userland
and kernel policy. A program extension or executable-name suffix is not, by
itself, a trust boundary.

## Networking

Outbound DHCP, DNS, ICMP, TCP, HTTP, and HTTPS support exists in varying
degrees. A limited TCP listener is available to system-privileged programs.
There is no general ingress firewall, and the listener API is not POSIX
sockets. Do not expose the sample HTTP server to an untrusted network.

HTTPS requires a valid clock, hardware entropy, and a trusted CA bundle.
Certificate verification failures should be treated as failures; do not
replace them with a plaintext fallback.

## Development practice

- Use disposable QEMU disks for experiments that may damage filesystem data.
- Run untrusted guest programs only in a VM you can throw away.
- Check third-party source licenses and keep their notices with the source.
- Report security problems privately to the project maintainer rather than
  publishing exploit details before a fix is available.

For the implementation details, see `kernel/security/`,
`kernel/syscall.c`, and the [syscall guide](../userland/syscalls.md).
