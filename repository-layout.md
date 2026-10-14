# Repository layout

CreateOS has two supported source layouts. Both use the same kernel build
interface and expect the userland, docs, and SDK at fixed paths under the
repository root.

## Private monorepo

The private CreateOS repository keeps all source in one Git repository:

- `kernel/`, `boot/`, `linker.ld`, and the top-level `Makefile` are the kernel
  and build/boot infrastructure.
- `rootfs_maker/` contains the guest userland, syscall compatibility library,
  host image tools, and root filesystem builder.
- `sdk/` contains app headers, libraries, examples, and build scripts.
- `docs/` contains kernel, userland, and SDK documentation.
- `build/` contains generated artifacts and is not source of record.

This layout has no Git submodule or clone requirement for an ordinary build.
The SDK source is tracked in the private repository.

To synchronize private sources into local public project folders, run
`scripts/sync-public.sh [PUBLIC_WORKSPACE]`. By default, it uses
`../CreateOS-public/` and creates the `CreateOS/`, `rootfs_maker/`, `Docs/`,
and `sdk/` directories there when missing. It performs local copies only,
replaces stale source files within these four exact folders, and preserves
`.git` metadata when present. It does not commit or push; review and publish
the changes with GitHub Desktop.

## Public kernel repository

The public `CreateOS` repository is intended to contain the kernel, boot
infrastructure, linker script, and top-level build scripts. Its
`scripts/assemble-project.sh` clones or fast-forwards the three component
repositories into `/rootfs_maker/`, `/docs/`, and `/sdk/` within the kernel
checkout:

| Directory | Repository | Required files |
| --- | --- | --- |
| `rootfs_maker/` | [CreateOS-Dev/rootfs_maker](https://github.com/CreateOS-Dev/rootfs_maker) | `Makefile`, `lib/user_syscall.h`, `root/prog/shell/shell.c` |
| `docs/` | [CreateOS-Dev/Docs](https://github.com/CreateOS-Dev/Docs) | `kernel/overview.md`, `userland/userland.md`, `sdk/app-development.md` |
| `sdk/` | [CreateOS-Dev/sdk](https://github.com/CreateOS-Dev/sdk) | `Makefile`, `app.mk`, `include/createos/uikit.h` |

The layout matches build references such as `make -C rootfs_maker` and
`rootfs_maker`'s relative access to `../kernel` and `../sdk`. The script
verifies each remote and required source files. Existing component clones are
updated only when their worktrees are clean and can fast-forward. Run
`scripts/assemble-project.sh --check` to validate existing clones without
fetching, or `--build` to clone/update the components and build the complete
OS. The public component repositories must be populated first; use
`scripts/sync-public.sh` from the private checkout and GitHub Desktop to
publish them.

## Local customization

Edit the private monorepo as the source of truth, then run
`scripts/sync-public.sh` to refresh the public checkout working trees. That
sync intentionally removes stale files inside the four exact public
repositories while preserving Git metadata. Do not keep uncommitted work only
in those generated source trees. Review and publish each resulting repository
with GitHub Desktop. Do not treat `build/` as a source directory or put
generated images in component repositories.
