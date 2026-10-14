# Notes for developers and coding assistants

This guide is for anyone editing CreateOS, whether they use an editor, a
terminal, or an AI coding assistant. The important thing is to follow the
interfaces already in the tree and test the result at the right layer.

## Before changing code

1. Start at the subsystem's header and implementation; follow one caller
   before adding a new interface.
2. Check whether the same behavior already exists in `kernel/`,
   `rootfs_maker/`, or `sdk/`.
3. Keep changes local to the request. Do not reformat unrelated files or
   replace project-specific code with a generic POSIX assumption.
4. The complete source tree contains every component. A kernel-only checkout
   obtains userland, docs, and SDK from their component repos.

## Build and test

For a kernel or guest-userland build, use the pinned local cross-toolchain:

```sh
make toolchain
make verify-toolchain
make
```

For an SDK change, build an example:

```sh
sdk/scripts/build-example.sh hello
sdk/scripts/build-example.sh uikit-demo
```

For a change to visible boot or guest behavior, try QEMU with `make run` or
`make serial`. Use a disposable disk for experiments. A successful compile
only proves that the compiler and linker accepted the code.

## Boundaries worth remembering

- The target is x86-64; guest programs are native ELF executables.
- CFS5 is a custom filesystem, not ext4 or FAT.
- The shell is a small command interpreter, not Bash.
- Guest TinyCC and Lua exist; host tools such as Python, Make, NASM, and
  `grub-mkrescue` are not automatically guest programs.
- The network TCP listener is a small CreateOS API, not POSIX sockets.
- `.APPL` applications are not sandboxed.
- A build from the complete source tree should not need to clone components.

When documenting a behavior, verify it in the current source or run it. Say
when something is planned or untested rather than presenting it as finished.
