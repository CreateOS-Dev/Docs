# Work in progress

This list is a guide to useful next steps, not a promise that every item has
an owner or a delivery date. If you pick something up, check the code and
open issues first; the implementation may have changed since this list was
written.

## Make it easier to test

- Add a small repeatable test suite for CFS5, syscall argument validation,
  user-process transitions, and the SDK HTTP parser.
- Add QEMU smoke tests for first boot, login, shutdown, and guest networking.
- Keep disposable disk images for tests so a test run never touches a
  developer's saved `build/disk.img`.
- Document the exact hardware and QEMU configurations used for each driver
  check.

## Kernel and hardware

- Exercise storage discovery on real AHCI and NVMe devices; QEMU coverage is
  not a substitute for different firmware and controllers.
- Add recovery and consistency checks for interrupted CFS5 writes.
- Continue testing framebuffer text, fonts, mouse input, keyboard layouts,
  USB HID, and timer routing across more than one emulator configuration.
- Review process cleanup and resource ownership when a program exits or is
  replaced.

## Networking

- Test malformed and fragmented packets, DNS failures, TLS certificate
  failures, and TCP connection cleanup in QEMU.
- Decide how inbound traffic should be filtered before exposing listener
  applications beyond a development network.
- Keep HTTP and TCP listener APIs small until process scheduling, polling,
  and descriptor ownership are ready for a larger server.

## Userland and SDK

- Improve shell parsing without claiming full POSIX shell compatibility.
- Add clearer error messages and recovery paths to first boot and account
  management.
- Grow the SDK examples alongside stable APIs and add host-side build checks
  for them.
- Port another guest language only when its runtime, filesystem access, and
  process behavior can be maintained and tested as part of CreateOS.

## Contributions

Before changing an area, read its guide in `docs/`, build the smallest target
that covers the change, and try it in QEMU if the behavior is visible at
runtime. Record what you actually tested; a successful compile is not a
runtime test.
