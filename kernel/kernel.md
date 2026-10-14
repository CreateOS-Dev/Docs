# Kernel and memory architecture

## Architecture

The current supported build target is x86-64. Low-level boot, CPU, interrupt,
syscall, APIC, and TSS code is kept under `kernel/arch/`; generic declarations
are under `kernel/header/`. A boot loader being able to load the image does not
make another CPU architecture supported. Runtime interrupt routing currently
uses the legacy PIC: IOAPIC routing is deferred because its timer delivery has
not yet passed periodic-tick tests in QEMU.

## Kernel subsystems

- PCI discovery provides device identities and BAR information to drivers.
- The console layer provides COM1 8250/16550 serial, VGA/framebuffer, and TTY
  facilities. COM1 is detected with a UART loopback check, transmit polling is
  bounded, and the TTY falls back from serial if the UART fails at runtime.
- Framebuffer text can use the bundled Noto Sans TrueType font through
  libschrift. UTF-8 output is decoded into Unicode code points before console
  cells are stored, and the display font can be changed in Settings. The font
  is embedded in the kernel and installed under `/usr/share/fonts`.
- TinyGL is built as a kernel-side software renderer. Kernel clients can use
  `framebuffer_tinygl_init()` and `framebuffer_tinygl_present()` from
  `kernel/header/drivers/tinygl.h`; the current adapter targets 32-bit
  framebuffers and uses TinyGL's 16-bit internal color buffer. TinyGL is not
  exposed as a userland graphics API.
- Clock/timer code supplies ticks used by timeouts and driver polling.
- The memory subsystem provides physical/virtual memory helpers and user
  address-space setup. The VMM can create uncached identity mappings for
  MMIO ranges across the low canonical address space, including PCI BARs above
  4 GiB, creating page-table levels and splitting existing 2 MiB mappings as
  needed.
- ACPI HPET tables are validated for system-memory GAS entries. A supported
  64-bit HPET counter provides monotonic nanoseconds; the PIT remains the
  interrupt and timeout source and is used when HPET is absent or unsupported.
- `kernel/proc/appl.c` loads the current user executable format; syscall code
  exposes kernel operations to user applications.
- The VFS dispatches file operations to filesystem implementations.
- Linux user applications, Linux kernel modules, and Linux driver interfaces
  are not supported. Linux application compatibility would require a Linux
  syscall ABI, ELF/interpreter and dynamic-library support, process/signal/
  memory semantics, and a compatible C runtime. Linux kernel drivers depend on
  Linux kernel internals and cannot be loaded as CreateOS drivers.

## Memory and process limitations

CreateOS is not a full process-isolated Unix kernel. Its executable loading,
address-space management, process control, fork/wait behavior, and resource
accounting remain limited and must be assessed against the implementation
before relying on them. The existence of a syscall declaration is not proof
that it has complete POSIX semantics.

The native `sys_run` syscall provides a synchronous child-image transition for
applications that need to run a command and inspect its exit status. It saves
and restores the caller's syscall frame, user address space, heap bookkeeping,
privilege state, and working directory, and permits at most eight nested
children. It does not implement `fork`/`wait`, per-process syscall stacks,
independent descriptors, or process-level isolation; transitions invalidate
console-scoped descriptors.

The console's Unicode path decodes UTF-8 output; it does not implement
system-wide UTF-16 text storage or bidirectional text layout. The Noto Sans
renderer currently uses fixed-width console cells and falls back to the
bitmap fonts if TrueType initialization or glyph rendering fails.

User mappings set write permission on intermediate page-table entries while
the leaf entry retains each page's requested permissions. This allows writable
user stack/data pages without making read-only executable pages writable.
Copy-on-write, VMA tracking, demand paging, and a page cache are not implemented.

## Extending the kernel

Put public interfaces in the relevant subdirectory of `kernel/header/` and
implementations in `kernel/`. Keep platform-specific operations behind
existing architecture or device interfaces. Add sources to the explicit
network source list or the normal kernel source discovery as appropriate, then
run `make verify-toolchain` and build the kernel before testing in QEMU.
