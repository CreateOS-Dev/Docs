# Security model and limitations

CreateOS is experimental. It is not suitable for protecting sensitive data or
exposing as a production service. The current user, syscall, memory, filesystem,
network, and driver implementations have not been established as a complete
security boundary.

Some code performs user-pointer validation, executable checks, stack
protection, and network input validation. These measures are useful but do not
constitute a complete audit or a guarantee against memory corruption,
privilege escalation, data loss, or denial of service.

The kernel stack-protector guard is initialized from RDSEED/RDRAND after CPU
feature detection. Boot stops if the hardware random source is unavailable or
returns an unusable guard; the kernel does not substitute a predictable
pseudorandom value. POSIX file descriptors and the persistent outbound TCP
stream are revoked or rejected across user-image transitions using a
per-console execution generation. These controls do not create independent
ring-3 processes: credentials, working directories, and other state remain
console-scoped, and applications replace one another rather than being
scheduled independently.

Development guidance:

- Run untrusted experiments in a disposable VM without shared folders or
  sensitive host devices.
- Do not attach a valuable physical disk or pass-through device.
- Treat malformed filesystem images, network frames, and syscall arguments as
  adversarial inputs and test bounded failure paths.
- Keep downloaded third-party code pinned and retain upstream license files.
- Report a feature as secure only after focused testing and review of the
  entire path, not because an individual check or driver exists.
