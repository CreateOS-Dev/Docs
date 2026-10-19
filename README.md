# CreateOS notes
CreateOS assisted with AI (Not developed with AI) 
These pages are written for people building, using, or changing CreateOS.
They focus on the behavior in the source tree today; this is still a
developing OS, so check the code when a detail matters.

## Start here

- [System tour](kernel/overview.md) — the boot path and where the code lives.
- [Build and run](kernel/build-and-run.md) — toolchain, QEMU, and disk images.
- [Use the userland](userland/userland.md) — shell, accounts, apps, and guest tools.
- [Build an application](sdk/app-development.md) — SDK setup and app projects.
- [C programming](sdk/c.md) — the app ABI, available headers, and examples.
- [UI Kit](sdk/uikit.md) — text-screen API and input helpers.
- [Lua](sdk/lua.md) — interpreter, modules, and guest-specific limits.
- [x86-64 assembly](sdk/assembly.md) — GAS apps and NASM boot code.
- [Shell commands](userland/shell.md) — entering commands and editing a line.

## Reference

- [Kernel notes](kernel/kernel.md)
- [Devices and storage](kernel/devices-and-storage.md)
- [Networking and toolchains](kernel/network-and-toolchain.md)
- [Security and trust](kernel/security.md)
- [System calls](userland/syscalls.md)
- [Supported languages](sdk/languages.md)
- [Source tree layout](repository-layout.md)
- [Developer notes](ai-development.md)
- [Work in progress](TODO.md)
