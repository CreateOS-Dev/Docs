# Build an application

CreateOS applications are static x86-64 ELF executables with the `.APPL`
suffix. The SDK in `sdk/` supplies the linker rules, startup code, syscall
library, and small UI/network libraries used by the included examples.

The compiler runs on macOS or Linux and targets CreateOS. An `.APPL` is not a
sandboxed package; only run code you trust.

## Build the examples

From the repository root:

```sh
make toolchain
./sdk/scripts/build-example.sh hello
./sdk/scripts/build-example.sh uikit-demo
```

The example Makefiles show the project pattern. The cross-toolchain defaults
to `build/toolchain/x86_64-elf/`; set `TOOLCHAIN_PREFIX` if yours is somewhere
else.

## Set up your own project

Create a directory with a `src/` folder and a Makefile:

```make
APP_NAME := notes
APP_VERSION := 0.1.0
APP_SRC_DIR := $(CURDIR)/src
SDK_ROOT ?= /path/to/CreateOS/sdk

include $(SDK_ROOT)/app.mk
```

Put one or more C sources in `src/`. The SDK discovers `.c` and uppercase
`.S` assembly files recursively, so you can organize them into subdirectories
without maintaining a source list. `APP_NAME` and `APP_VERSION` are also
available to C code as string macros.

Build with:

```sh
make -C /path/to/notes \
  SDK_ROOT="$PWD/sdk" \
  TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf"
```

The output is placed below `build/` (the default depends on whether the SDK
is used from the monorepo or a standalone checkout). You can set `BUILD_DIR`
to choose another output directory.

Projects that do not use a `src/` directory can set `APP_SRC` to one C file
and `APP_OBJECTS` to additional prebuilt objects.

## Using the SDK libraries

Include public headers from `sdk/include/createos/`:

- `<createos/uikit.h>` provides the current text UI helpers for screen
  dimensions, colored text, keyboard input, mouse position, and full-screen
  lifecycle.
- `<createos/network.h>` exposes the limited TCP listener and stream API
  available to system-privileged apps.
- `<createos/http.h>` contains bounded HTTP parsing helpers; it is not a
  complete HTTP client or server.

The SDK builds and links its static libraries automatically. The UI Kit is a
console API, not a window manager. See the [UI Kit reference](uikit.md) for
each function and a complete example.

## Add the app to the OS image

To stage an app for a locally built image, either copy the finished file into
`rootfs_maker/root/appcustom/` or use the project's install target:

```sh
make -C /path/to/notes install \
  SDK_ROOT="$PWD/sdk" \
  TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf" \
  CUSTOM_APP_DIR="$PWD/rootfs_maker/root/appcustom"
```

Then stage the userland and build the image:

```sh
make userapps
make disk-image
make
```

Custom apps are placed in `/app/custom/` in the guest and can be opened from
Explorer. The direct-copy instructions are in
`rootfs_maker/root/appcustom/README.md`.

## Existing projects

The built-in apps are separate projects under `rootfs_maker/root/app/`.
Each has its own Makefile and is discovered by the userland build. To build
one app while working in the monorepo:

```sh
make -C rootfs_maker/root/app/explorer
make -C rootfs_maker/root/app/textedit
```

The Lua and HTTP sample projects have additional notes in their source
directories. The [Lua guide](lua.md) documents the guest interpreter and
bytecode compiler. The guest has TinyCC as well, but the host SDK remains the
recommended way to build an app for a particular checkout's ABI.

## Limits

The SDK currently builds C and x86-64 assembly. It does not compile C++, Rust,
Go, or other guest languages. It does not provide dynamic linking, a window
system, or a general-purpose POSIX runtime. Keep dependencies small and
verify any syscall your app relies on against the kernel in the same checkout.
For the SDK's assembly format, see the [x86-64 assembly guide](assembly.md).
The [C guide](c.md) explains the available user library and app ABI in more
detail.
