# AI-assisted CreateOS development

This document gives coding assistants and maintainers a concise, verified
map of the project. It describes implemented behavior, not a promise that
every planned feature is available.

## Source-of-truth layout

- `kernel/`, `boot/`, `linker.ld`, and the top-level `Makefile` define the
  x86-64 kernel, boot process, and build orchestration.
- `rootfs_maker/` contains guest applications, commands, syscall wrappers,
  filesystem image tools, and root filesystem assembly.
- `sdk/` contains the application build rules, headers, libraries, examples,
  and project-generation scripts.
- `docs/` documents the kernel, guest userland, SDK, learning paths, and
  repository layout.
- `build/` contains generated or downloaded build inputs; it is not a source
  directory.

The private repository is a self-contained monorepo. Ordinary private builds
must not require Git submodules or cloning other repositories. The public
kernel repository downloads `rootfs_maker`, `docs`, and `sdk` into local
directories which its `.gitignore` excludes. Use
[`repository-layout.md`](./repository-layout.md) for update and bootstrap
commands.

## Guest language support

| Language/tool | Runtime/target status |
| --- | --- |
| C / TinyCC | Static x86-64 guest executables using CreateOS headers and syscalls. No dynamic linker or `tcc -run`. |
| Lua 5.4 | Guest interpreter and bytecode compiler are present; native C modules are disabled. |
| x86-64 assembly | Supported as kernel/startup source and SDK `.S` files. |
| Bash/sh, Python | Host build/configuration tools only; not guest runtimes. |
| C++, Rust, Go, Java, JavaScript | Not currently provided as guest toolchains/runtimes. |

See [`sdk/languages.md`](./sdk/languages.md) for beginner examples and
porting requirements. Do not claim guest support based only on a host
compiler or downloaded source archive.

## Guidance for coding agents

1. Read the relevant subsystem documentation and inspect the existing
   implementation before editing. Keep changes scoped and preserve the
   syscall and executable ABI.
2. Distinguish host tools from guest programs. Guest code must use the
   repository's freestanding headers and syscall wrappers, not assume Linux,
   libc, POSIX process scheduling, or a dynamic loader.
3. Prefer existing vendored and pinned dependencies. Before adding one,
   establish the supported guest use case, source version, upstream digest,
   license, build target, and storage location. Fetch source into `build/`,
   never silently replace a source checkout, and do not download arbitrary
   "all-language" bundles.
4. Report unsupported runtime features honestly. A dependency compiling on
   macOS or Linux does not demonstrate that it runs on CreateOS.
5. Validate the exact changed surface: shell syntax for scripts, a focused
   target for changed C/assembly, an SDK example for build-rule changes, and a
   complete build or QEMU run when the required local toolchain is available.
6. Preserve persistent guest data. `build/disk.img` is not a disposable build
   intermediate; do not clean or recreate it without an explicit need.

## Useful entry points

```sh
scripts/sync-public.sh
scripts/assemble-project.sh --check
scripts/assemble-project.sh --build
sdk/scripts/build-example.sh hello
sdk/scripts/build-example.sh uikit-demo
make check-build-tools
make verify-toolchain
```

The private monorepo is self-contained. `scripts/sync-public.sh` copies its
kernel, userland, documentation, and SDK sources into the four local public
project folders, replacing stale files while preserving any `.git` metadata.
It performs only local file operations; publish the updated repositories with
GitHub Desktop. From the public kernel checkout,
`scripts/assemble-project.sh` combines sibling userland, docs, and SDK folders
into the paths needed to build the complete OS.
