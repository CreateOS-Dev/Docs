# Development roadmap

This roadmap tracks unimplemented work and test gaps. Items are not represented
as complete merely because related source files exist.

## Build and userland

- [x] Add a system Settings `.APPL` for implemented keyboard, cursor, and DHCP
      autostart controls; surface write failures and omit unsupported settings.
- [x] Warn before Explorer executes native ELF binaries that are not `.APPL`;
      keep the prompt explicitly informational, not a security boundary.
- [x] Add bounded, per-console user keyboard layout files below
      `~/.config/keyboards/` with a validated active-layout selection.
- [ ] Run reproducible builds with the pinned cross-toolchain on macOS and
      Linux; verify the toolchain probe in both environments.
- [ ] Add shell parser tests, then implement and test quoting, script
      execution, status propagation, and redirection as distinct features.
- [x] Add a bounded synchronous `sys_run` child-image API with nested execution,
      exit-status propagation, parent address-space/heap/privilege/CWD restore,
      and QEMU coverage for success, failure, and missing executables.
- [ ] Complete the ring-3 process lifecycle before treating guest build tools
      as general-purpose: independent register/syscall stacks, credentials,
      file descriptors, process ownership, concurrent scheduling, signals,
      and process groups. `fork()`/`wait()` currently deep-copy memory but run
      a child only when its parent waits; `sys_run` is also synchronous.
- [ ] Run the new deep-copy user-address-space isolation check in QEMU, then
      add QEMU execution coverage for the fork/wait return values, wait status,
      and restoration of parent memory, CWD, and umask.
- [ ] Evaluate and port a guest libc (mlibc or another explicitly selected
      implementation) against the process and filesystem APIs; cover its
      headers, syscalls, file I/O, memory functions, and static linking with
      tests.
- [ ] Finish QEMU validation of pinned TinyCC as a CreateOS-hosted userland
      compiler. Its source is vendored under `kernel/lib/third_party/tinycc`
      but must not be linked into the kernel; its platform adaptation and
      CreateOS headers, startup objects, and static runtime are built for the
      guest and installed in the rootfs. The `.APPL` and `/bin/tcc.bin` wrapper
      use the existing ring-3 executor and kernel syscalls. Compile, link, and
      execute a static C smoke program in QEMU before treating guest `tcc` as
      runtime-verified.
- [ ] Implement a guest `make` subset after guest `cc` is available. Run each
      recipe with `sys_run`; cover Makefile parsing, dependencies, recipe
      failures, and exit-status propagation; never report a failed build as
      successful.
- [ ] Enable TextEdit to detect C projects and offer Makefile generation/build
      only after the guest `make` and `cc` commands pass their runtime tests.
- [ ] Design and implement Linux userspace compatibility only after CreateOS
      has the required ELF interpreter/dynamic loader, process, memory, signal,
      filesystem, and syscall semantics. Linux kernel modules and drivers are
      not binary-compatible with CreateOS drivers.
- [ ] Recreate a maintained test suite for framebuffer cursor persistence,
      theme switching, and pointer movement after redraw; the repository
      `tests/` directory was removed.
- [ ] After guest process and filesystem APIs are ready, provide guest
      assembler/linker support and a GNU `make` subset; validate assembling,
      linking, recipe failures, and status propagation. GRUB's `grub-mkrescue`
      remains a host ISO-creation tool unless a guest image-writing workflow is
      separately designed and tested.
- [ ] Add an explicitly bootable compressed-image path with a tested XZ
      decompressor before replacing the current uncompressed Multiboot2
      kernel/rootfs boot path. `make compressed-release` currently creates
      only a verified distribution archive.
- [ ] Expand POSIX headers and runtime functions with conformance-oriented
      tests; do not claim complete POSIX compatibility until those tests pass.
- [ ] Automate inbound QEMU TCP tests for the admin-gated listener API,
      covering accepted-stream ownership, process-transition cleanup, and
      denial of
      unowned listener/stream handles; manual `httpd.APPL` testing confirmed
      one forwarded `GET /health` returns HTTP 200, but do not treat listening
      as a firewall.
- [x] Add a bounded HTTP/1.0/1.1 request parser to the guest app SDK with host
      tests for partial requests and ambiguous/malformed headers. This parser
      does not itself serve HTTP or validate inbound TCP behavior.
- [x] Add a single-request HTTP diagnostic `.APPL` to exercise the listener
      API and request parser; it is not a general-purpose server or Nginx port.
- [ ] Port Nginx as a native guest app only after guest libc/filesystem support,
      POSIX socket/event-polling interfaces, and tested inbound traffic controls
      exist; verify HTTP serving and shutdown in QEMU before shipping.
- [ ] Extend the cooperative x86-64 kernel-task context switch with a
      per-process ring-3 syscall stack, credentials, CWD, file descriptors,
      and capabilities before scheduling user applications.
- [x] Reserve a ring-3 address window from 1–2 GiB, remove that window from
      each user address space's kernel identity map, and reject user mappings
      outside it. This prevents virtual-address aliasing; it is not complete
      process isolation.
- [x] Add a QEMU self-test for cooperative kernel-task switching, address-space
      selection, user/kernel mapping separation, unauthenticated privileged
      syscall denial, cross-page user-string rejection, execution-generation
      descriptor revocation, active-session login rejection, and task cleanup;
      run it before filesystem initialization.
- [x] Require administrator credentials for user-triggered reboot/shutdown
      and for outbound ICMP, DNS, and TCP client syscalls. This is a coarse
      account gate; per-process opt-in capabilities and packet filtering remain
      outstanding.
- [x] Require the active executable to originate from a protected system
      binary directory before honoring administrator-only syscalls. This is a
      path-based boundary, not a signed executable or a process capability.
- [x] Prevent user-requested `exec` from gaining system privilege only by
      selecting a system-directory executable; keep trusted kernel recovery
      transitions distinct from application-controlled exec.
- [x] Keep exit and user-fault recovery from restoring system privilege based
      only on the recovery shell's path. Restore parent-process privilege only
      after the kernel stores and validates parent ring-3 contexts.
- [x] Prevent an authenticated app from replacing its console's active user
      through the login syscall; add non-mutating current-password verification
      for `passwd`. Disable cross-account `sudo` until credentials can be
      scoped to processes.
- [x] Bind POSIX descriptors and TCP client streams to an execution generation;
      revoke prior handles when a user image is replaced and require exact TCP
      stream handles for read, write, and close.
- [ ] Add a QEMU networking test proving that an open TCP stream cannot be
      reused after a user-image transition and that the previous stream closes;
      also check that stream handles always fit the positive syscall return
      range.
- [x] Assign non-root accounts distinct filesystem UIDs, migrate each existing
      CFS home tree to its account on login, and set owners on new home files
      and directories.
- [ ] Add QEMU coverage for home ownership migration, new-path ownership, and
      cross-account filesystem denial before treating that boundary as verified.
- [ ] Add an SVG parser/renderer with bounds, resource, and malformed-input
      tests before exposing an SVG viewer command.

## Networking and drivers

- [x] Smoke-test E1000/lwIP DHCPv4, DNS, ICMP echo, and TCP HTTP in QEMU user
      networking; fix syscall argument marshalling and raw ICMP header parsing.
- [ ] Add repeatable packet-level lwIP tests for timeout behavior and malformed
      frames; smoke tests do not establish protocol conformance.
- [x] Add the legacy PCI VirtIO-net transport behind `netdev`; QEMU boot
      and user-mode tests confirm DHCPv4, ICMP echo, DNS, and HTTP.
- [x] Start DHCP automatically before login by default; allow disabling it
      with `/etc/network.conf` and retain manual DHCP commands.
- [x] Add the modern PCI VirtIO-net transport and validate DHCPv4 DORA with a
      QEMU modern-only device. Legacy and modern header sizes are handled
      separately.
- [x] Verify modern VirtIO-net ICMP echo, DNS, and HTTP in QEMU user-mode
      networking with a disposable disk; confirm ICMP echo reply, DNS A
      response, and `HTTP/1.1 200 OK` in serial output and packet capture.
- [x] Add a Mbed TLS 4.0.0 HTTPS client to `curl`, constrain it to TLS 1.2/1.3,
      require certificate-chain and hostname validation, and fail closed on
      unavailable RTC or hardware entropy.
- [ ] Validate HTTPS success with a trusted certificate and rejection of an
      untrusted certificate, mismatched hostname, and invalid clock in QEMU;
      repeat the existing HTTP smoke test after TLS integration.
- [x] Add a bounded `curl -o` path that validates response status/framing,
      holds the response in application memory, and requires explicit user
      confirmation before saving below the user's home; ask separately before
      replacing an existing file.
- [ ] Expand downloads beyond the 4 KiB response limit and support chunked
      transfer encoding with partial-response, storage-failure, and TLS tests.
- [ ] Add packet-level deny-by-default inbound tests for every network
      transport, preserving only validated replies to guest-initiated traffic;
      the current stack has no enabled listener, but is not a general firewall.
- [ ] Add a Realtek RTL8111 PCIe driver; do not treat RTL8139 support as
      RTL8111 compatibility.
- [x] Collect PCI IDs for the TERRA MOBILE 360-11 V3. Jasper Lake devices
      include xHCI `8086:4ede`, Serial IO I2C `8086:4ee8`, HDA `8086:4ec8`,
      UHD graphics `8086:4e71`, and AX201 `8086:4f30`. Exact I2C-HID/ACPI
      child-device resources, USB inventory, and battery identifiers are still
      needed before selecting those drivers.
- [ ] Validate E1000 and RTL8139 on target hardware.
- [ ] Port additional upstream drivers only with license review, CreateOS
      adaptation, error handling, and test coverage.
- [ ] Add audio device discovery, DMA/interrupt handling, codec support, a
      playback API, and QEMU/real-hardware validation.
- [ ] Validate HPET monotonic-clock stability and conversion against a known
      QEMU clock and real hardware; PIT remains the interrupt time source.
- [ ] Add an Intel Jasper Lake Serial IO I2C driver for `8086:4ee8` and a
      tested transfer API; PCI MMIO mapping now supports its 64-bit BAR above
      4 GiB.
- [ ] Add ACPI device enumeration and I2C-HID support for laptop touchpads;
      the current PS/2 and USB boot-mouse paths do not cover modern
      touchpads.
- [ ] Identify target Wi-Fi chipsets from PCI/USB IDs, then implement a
      firmware-aware WLAN driver and authentication/network management.
- [ ] Add Bluetooth USB HCI transport, controller firmware loading, and an
      HCI host stack; PCI enumeration alone is not Bluetooth support.
- [ ] Add ACPI battery/charger reporting backed by validated AML/EC access,
      and test charge state, capacity, and unplug/plug changes on hardware.
- [ ] Add USB UVC camera and microphone input support with device discovery,
      streaming, and permission-aware user APIs.
- [ ] Add display-controller modesetting, EDID, and hot-plug support for HDMI
      and multiple screens; the boot framebuffer is not multi-monitor support.
- [ ] Add Intel HD Audio controller/codec support, microphone capture, and
      jack detection; PC-speaker beeps and AC'97 probing are not a sound stack.
- [ ] Add AC'97 codec/DMA playback support and QEMU/real-hardware tests.

## Storage and USB

- [x] Exercise xHCI BOT single-sector write/read and restoration against a
      disposable QEMU USB image, checking that the image is unchanged afterward.
- [ ] Add repeatable tests for block devices, GPT/MBR boundaries, FAT32/CFS5,
      mount/unmount, read/write failures, and removal.
- [ ] Expand xHCI storage beyond one full-/high-/SuperSpeed BOT device per
      controller; exercise hotplug and timeout/error paths in QEMU and on
      hardware before claiming USB storage compatibility.
- [x] Add repeatable QEMU enumeration tests for xHCI boot-protocol USB HID
      keyboards and mice.
- [ ] Validate injected USB key/mouse events end to end and support multiple
      USB slots per xHCI controller so HID and storage can coexist.
- [x] Add host-side USB HID report parser tests and reject malformed keyboard
      usages and report sizes.
- [ ] Implement and test an installer against disposable virtual disks before
      attempting real-PC installation.
- [ ] Validate GRUB Multiboot2 boot from removable media on real UEFI and
      legacy-BIOS x86-64 PCs; document Secure Boot requirements and recovery
      steps before claiming general PC boot support.

## Kernel and security

- [ ] Define tested architecture/hardware support; the current kernel target is
      x86-64, not every architecture.
- [ ] Implement and test ten independent virtual terminals with separate
      backing buffers before advertising multi-terminal support.
- [ ] Add LAPIC application-processor startup and per-CPU GS state, then test
      cross-CPU scheduling and interrupt delivery under QEMU SMP.
- [ ] Diagnose why IOAPIC-routed PIT interrupts stop advancing the kernel tick
      in QEMU, then enable APIC routing only after periodic-tick and DHCP tests
      pass.
- [ ] Add XSAVE/XRSTOR context management and timer-driven preemption only
      after interrupt-frame switching, per-task kernel stacks, and SMP-safe
      scheduler tests are in place; add sleep/wakeup, affinity, and reschedule
      IPI tests.
- [ ] Expand physical and virtual memory with slab allocation, copy-on-write,
      VMA tracking, demand paging, and a page cache, each with fault and
      concurrency tests.
- [ ] Add negative tests for syscall pointer validation, executable loading,
      filesystem metadata, driver input, and resource exhaustion.
- [x] Seed the kernel stack-protector guard with RDSEED/RDRAND and halt boot if
      hardware entropy is unavailable; do not fall back to a predictable guard.
- [ ] Implement process-owned resources and enforced filesystem/network
      capabilities, then add isolation tests for system-write denial, private
      app data, and opt-in networking. The current cooperative kernel-task
      scheduler does not isolate applications; console-scoped UIDs and
      credentials and working directories are not process isolation. Ring-3
      images remain non-preemptive, and forked children currently run only
      after a parent waits for them.
- [x] Implement and pass the QEMU ACPI S3 enter/wake/resume test, including
      legacy real-mode and advertised 64-bit wake-vector entry points.
- [ ] Validate keyboard and power-button wake sources and the automatic idle
      policy on real hardware. QEMU's `system_wakeup` test does not establish
      those hardware wake paths; the idle poll currently runs while interactive
      input is waiting. Default policy is 15 minutes.
- [ ] Verify that `/etc/keyboards/default`, `us.lyt`, and `fr-azerty.lyt` are
      included and selected correctly; add QEMU scancode tests and complete
      French AZERTY behavior, including accents and modifier combinations.
- [x] Apply supported keyboard-layout and cursor-theme changes immediately
      after a successful `/etc` setting update; preserve interrupt state while
      replacing the active keyboard map.
- [ ] Define and package English/French application translations, connect
      locale selection to applications without conflating it with keyboard
      layout, and test fallback behavior for missing translations.
- [ ] Apply `/etc/timezone` to clock display/conversion with a documented
      timezone database or explicitly supported offset set, including daylight
      saving transitions and UTC-based RTC tests.
- [ ] Add focused QEMU interaction tests for Settings persistence, keyboard
      remapping, cursor-theme changes, and mouse movement at screen edges.
- [ ] Validate that kernel, userland, and rootfs version headers consistently
      wrap the canonical `kernel/header/version.h`; add compile-time and
      runtime `uname`/version ABI checks.
