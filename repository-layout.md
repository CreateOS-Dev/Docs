# How the source tree fits together

CreateOS can be kept in one complete source tree or split into four
component repositories. The kernel build expects the same directory layout
either way.

## Complete source tree

The full tree contains everything needed to build an image:

- `kernel/`, `boot/`, `linker.ld`, `configs/`, and the root `Makefile` build
  the kernel and boot image.
- `rootfs_maker/` builds guest programs and the CFS5 root filesystem.
- `sdk/` contains application headers, libraries, build rules, and examples.
- `docs/` contains the guides.
- `build/` holds generated output and is not source.

An ordinary build of this tree does not need to clone any component repo.

## Four local project folders

To copy the complete tree into four separate working folders, run
`scripts/sync-public.sh` from the repository root:

```sh
scripts/sync-public.sh
```

The default destination is `../CreateOS-public/`:

```text
CreateOS-public/
├── CreateOS/
├── rootfs_maker/
├── Docs/
└── sdk/
```

This is a local file sync. It does not run Git commands or access a network.
Source files in those four destination folders are replaced to match the
complete tree, while any `.git` metadata is left in place. Keep edits in the
complete tree, then review and publish each folder with GitHub Desktop.

Use a different destination by passing it as the first argument:

```sh
scripts/sync-public.sh /path/to/CreateOS-projects
```

## Build from the kernel repo

The kernel Makefile expects these component repos alongside it inside the
kernel checkout:

| Directory | Component repo |
| --- | --- |
| `rootfs_maker/` | `CreateOS-Dev/rootfs_maker` |
| `docs/` | `CreateOS-Dev/Docs` |
| `sdk/` | `CreateOS-Dev/sdk` |

Run `scripts/assemble-project.sh` in the kernel repo to clone a missing
component or fast-forward a clean existing checkout. It uses Git for that
operation. It will not overwrite an unrelated directory, a dirty checkout,
or a checkout with an unexpected remote.

```sh
scripts/assemble-project.sh
scripts/assemble-project.sh --check
scripts/assemble-project.sh --build
```

`--check` validates existing checkouts without fetching. `--build` updates
the components and then builds the OS. The normal kernel build also prepares
components before compiling.

## Layout matters

The userland Makefile uses paths such as `../kernel` and `../sdk`, so these
folders must sit beside the kernel directory in the assembled tree. Moving
one component without updating those references will break app builds.
Generated disk images belong under `build/`, never inside a source component.
