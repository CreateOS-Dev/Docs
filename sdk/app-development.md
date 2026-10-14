# CreateOS application development

## Built-in applications

Each built-in application is an independent project under
`rootfs_maker/root/app/`: `explorer/`, `settings/`, and `textedit/`. Each
directory contains its own `main.c` and `Makefile`; the Makefile includes the
shared rules from `sdk/app.mk`.

TextEdit includes a navigable file-tree sidebar, toggled with Ctrl+B. Launch it
without arguments to browse the current working directory, or pass a directory
to open that folder as the workspace:

```sh
open TextEdit.APPL
open TextEdit.APPL /home/user/project
```

Relative file paths resolve against the current directory (or workspace folder
when opening additional files). Select a file in the tree and press Enter to
open it. Ctrl+R runs only `.lua` files, using the embedded Lua runtime so
unsaved edits are included. On a `.c` file, Ctrl+R detects an existing
`Makefile`/`makefile`, or creates a non-overwriting CreateOS SDK Makefile
template in the source directory. The generated file uses
the C source basename as `APP_NAME` and a placeholder `SDK_ROOT`; set it to
the source checkout's `sdk/` directory when building on a host. TextEdit does
not run it yet: guest `make` and `cc` are not installed. Python execution is
likewise unavailable until a guest Python runtime is ported.

Build one application with:

```sh
make -C rootfs_maker/root/app/explorer
```

Build and stage all system applications with:

```sh
make userapps
```

## Build and add a third-party application

Create a project directory with a Makefile and place its C (`.c`) and
x86-64 assembly (`.S`) files under `src/`. The shared rules discover and
compile sources recursively:

```make
APP_NAME := hello
APP_VERSION := 0.1.0
APP_SRC_DIR := $(CURDIR)/src
SDK_ROOT ?= /path/to/CreateOS/sdk
include $(SDK_ROOT)/app.mk
```

Build the application using the SDK and matching cross-toolchain:

```sh
make -C /path/to/hello SDK_ROOT="$PWD/sdk" \
    TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf"
```

The built `.APPL` is a native x86-64 ELF executable. To add it to the
userland, either drag it into `rootfs_maker/root/appcustom/` or run the
project's `make install` target. Then stage it into the root filesystem and
refresh the disk image:

```sh
make userapps
make refresh-system-apps
```

The SDK includes two runnable source examples:

```sh
./sdk/scripts/build-example.sh hello
./sdk/scripts/build-example.sh uikit-demo
```

`hello` demonstrates a minimal UI Kit screen and key input; `uikit-demo`
demonstrates clearing, centered and positioned text, row filling, and key
handling. Set `TOOLCHAIN_PREFIX` when the pinned CreateOS cross-toolchain is
not in this checkout's `build/` directory.

Applications link to the OS syscall library and CreateOS UI Kit so builds
match the kernel ABI; the build does not download arbitrary third-party
libraries.

Include `<createos/uikit.h>` to use `createos_ui_init`,
`createos_ui_write_at`, `createos_ui_write_centered`,
`createos_ui_fill_row`, `createos_ui_clear`, `createos_ui_read_key`,
`createos_ui_mouse_position`, and `createos_ui_shutdown`. The current UI Kit
is a console/TUI helper, not a window manager or graphics API. Applications may
also include `user_syscall.h` for the filesystem, process, and system-call
interfaces. The SDK's static syscall, startup, and UI libraries are built
from the source checkout and are linked into each application.

`sys_run(path, argv)` synchronously loads a native executable as a child image,
then returns its signed exit status to the caller. It supports up to eight
nested calls and restores the caller's address space, heap bookkeeping, working
directory, and privilege state. It remains a separate API from POSIX
`fork`/`wait`.

The current `fork()`/`wait()` subset deep-copies the caller's mapped user pages.
The parent receives a child PID; when it calls `wait()`, the saved child
continues after `fork()` with return value `0`, and the parent resumes with the
child PID and POSIX-style wait status after the child exits. Up to four
unreaped children are retained system-wide, and pending child images are
released when their parent exits. This is deliberately serialized:
the child is not scheduled concurrently, there is no timer preemption, and
file descriptors remain console-scoped rather than being independently
duplicated. Programs must wait for a child before expecting its code to run.
The wait/child-state handling is in the kernel `SYS_WAIT` syscall; the
userland `wait()` function only adapts that syscall to POSIX return values and
`errno`. This is not complete POSIX process isolation.

The ELF loader maps loadable segments directly into separately allocated pages
and reads file data in 16 KiB chunks, so it does not require a contiguous
physical allocation proportional to the executable's size. The user image has
a 512 MiB virtual range; each application also has an 8 MiB stack and a
separate 504 MiB virtual heap reservation. These are address-space limits, not
guaranteed physical allocations: RAM must still be available for the mapped
image pages, stack, heap pages actually used, and page tables. The allocator
uses a next-fit search so loading many pages does not repeatedly rescan memory
from its beginning.

The file itself remains bounded by CFS5's 32-bit file-size field and available
disk space. CFS5 stores a file in at most eight extents, so a highly fragmented
disk can still prevent a large executable from being stored even when its
loadable segments fit the virtual image range. The ELF file may also exceed
512 MiB if its loadable segments fit; the filesystem and format limits, rather
than a single contiguous image buffer, determine that case.

Ctrl+C interrupts a foreground application while it polls terminal input or
waits in a blocking CreateOS network call, then returns to the shell. An
application that uses Ctrl+C as an in-app shortcut can disable this behavior
with `sys_tty_ctrl_c_interrupt(0)` and restore it with
`sys_tty_ctrl_c_interrupt(1)`. The default is restored whenever a new program
is started.

The compiler writes `build/rootfs_maker/app/<name>.APPL`. The application is
a native x86-64 ELF executable, not a sandboxed package; only install software
you trust.

From the shell, a binary in the current directory can be launched with
`./name`. When that exact path is absent, the shell tries `./name.bin`;
explicit `.bin` paths are used as written. Launch APPL applications with
`open name.APPL`. The `open` command accepts relative APPL paths in the
current directory as well as installed application names and paths.
Paths containing `/` are resolved relative to the current working directory
unless they begin with `/`, and are not searched in the system command
directories.

Files created by a compiler are not implicitly trusted or marked executable.
After generating a native ELF binary, mark it executable with `chmod +x
./name` (or set an explicit mode such as `chmod 755 ./name`), then run it with
`./name`. The command supports octal modes and symbolic execute-only clauses,
including `u+x`, `a-x`, and combinations such as `u+x,g+x`. A regular user may
change permissions only on files they own; admin permissions are required to
change another user's files. GCC is not included as a CreateOS guest command.

The POSIX compatibility layer also exposes `mkdir`, `unlink`, `remove`,
`rename`, `chdir`, and `chmod` over the existing filesystem syscalls. Their
success values follow the POSIX convention (`0`, even where an underlying
syscall returns a positive success value). `unlink` and `remove` reject
directories rather than deleting them; a POSIX `rmdir` interface with
empty-directory checks is not implemented yet. `access(path, X_OK)` checks
execute permission without opening the file for reading, and `umask(mask)`
filters permissions on newly created files and directories. Each console
starts with mask `022` after boot and retains it because CreateOS does not yet
have independent POSIX process state. A synchronous `sys_run` child inherits
the caller's mask, and changes made by the child are restored when it returns.

The repository does not currently include a maintained host or guest test
suite. Build the application with its Makefile, then install it in a disposable
CreateOS disk image and run it in QEMU. The missing automated coverage is
tracked in the [development roadmap](./TODO.md).

## Add a built-in app

Add `rootfs_maker/root/app/<name>/main.c` and a project Makefile that sets
`APP_NAME` and `APP_SRC`, then includes the shared `app.mk`. The top-level
rootfs build discovers these per-app Makefiles, builds each application
independently, and installs built-ins under `/app/system/bin/`.

## Drop-in custom applications

Copy or drag `.APPL` files directly into `rootfs_maker/root/appcustom/`.
Running `make userapps` stages them at `build/rootfs/app/custom/`; Explorer
launches them from `/app/custom/` in the next disk image.

To install an app from within CreateOS, open its directory in Explorer and drag
the `.APPL` file onto the highlighted install bar. This copies native ELF64
x86-64 applications to the current user's `~/.local/apps/` directory and does
not overwrite an existing app. Press `A` in Explorer to browse and launch
applications in that personal directory. A removable volume must be mounted
and opened in Explorer before its app can be installed. `.APPL` files are
native executables, not sandboxed packages; install only software you trust.

The SDK's headers and build rules are kept in the repository under `sdk/`.
Cross-compilation runs on the host and requires the matching CreateOS
cross-compiler.

System applications that need the current TCP server primitive can include
`<createos/network.h>` and link `libcreateos-network.a`. It exposes
`createos_net_tcp_listen`, `createos_net_tcp_accept`, stream read/write/close,
and listener close. These calls are blocking, support only one listener and
one accepted stream at a time, and require an authenticated administrator
system application. They are not POSIX sockets and are insufficient for
Nginx's event-driven worker model; the implementation is an OS networking
foundation, not an Nginx port.

Applications can also include `<createos/http.h>` and link
`libcreateos-http.a` for a bounded HTTP/1.0/1.1 request parser. It accepts
origin-form targets, validates header syntax, enforces a single
`Content-Length`, rejects `Transfer-Encoding` and folded headers, and requires
one non-empty `Host` header for HTTP/1.1. The parser reports incomplete input
separately; callers must enforce their own connection, request-body, and
timeout limits. The parser currently has no maintained automated test suite;
do not treat successful compilation as validation of malformed or partial
requests.

## Lua interpreter and bytecode compiler

The Lua 5.4.8 interpreter is a separate built-in app under
`rootfs_maker/root/app/lua/`; its Makefile compiles the vendored upstream source
in `kernel/lib/third_party/lua-5.4.8/` for CreateOS. Use `lua` for `-v`, `-e`,
script-file execution, or the interactive prompt. Its base, coroutine, debug,
I/O, math, OS, package, string, table, and UTF-8 libraries are opened. Lua
source modules and file I/O use the CreateOS filesystem compatibility layer.
Native C-module loading and subprocess execution are unavailable because the
kernel does not yet provide those process/runtime interfaces.

The same project builds the upstream bytecode compiler as `luac.APPL`; `luac`
is a shell launcher for that standalone application. For example:

```text
luac -o program.luac program.lua
lua program.luac
```

Build the interpreter, bytecode compiler, and command launchers with
`make userapps`. These are Lua tools, not a guest C compiler; `cc` remains
unavailable in CreateOS.
