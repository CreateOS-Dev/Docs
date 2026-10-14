# Learning CreateOS programming languages

This guide distinguishes languages that are usable in the guest OS from
languages used only on the development host. A compiler being available on the
host does not mean its programs can run on CreateOS.

## What runs in the guest today

| Language | Guest status | Use |
| --- | --- | --- |
| C | Native x86-64 applications are supported; guest TinyCC is installed as `/bin/tcc.bin` and `/app/system/bin/tcc.APPL`. | Compile a static native program using CreateOS headers and libraries. |
| Lua 5.4 | Lua and `luac` are installed as `.APPL` applications. | Run scripts with `lua file.lua`; compile bytecode with `luac`. |
| x86-64 assembly | Used for kernel, startup, and low-level SDK components. | `.S` files in an SDK application's `src/` tree are assembled by the cross-toolchain. |
| Shell | Host Bash/sh scripts build and package the OS. | The guest command shell is not a POSIX shell interpreter and does not execute `.sh` files. |
| Python | Used by host-side configuration/build utilities. | No guest Python runtime is included. |
| C++, Rust, Go, Java, JavaScript, and other runtimes | Not currently supported as guest toolchains/runtimes. | Host availability does not provide guest ABI/runtime support. |

The guest compiler does not implement dynamic linking or TinyCC's `-run`
execution mode. Compile a C program to a static executable, then launch the
result as a separate program:

```sh
tcc -o /home/root/hello hello.c
chmod u+x /home/root/hello
/home/root/hello
```

Compiler input and output paths follow CreateOS's filesystem and permission
rules. Only compile source code you trust; generated native binaries execute
with the caller's guest privileges.

## Learn C with the SDK

The SDK contains two buildable examples:

```sh
./sdk/scripts/build-example.sh hello
./sdk/scripts/build-example.sh uikit-demo
```

For your own project, create a Makefile that sets `APP_NAME` and
`APP_SRC_DIR`, then includes `sdk/app.mk`. C sources and uppercase `.S`
assembly files below `src/` are discovered recursively. Build products are
kept under `build/`; use `make install` or copy the finished `.APPL` into
`rootfs_maker/root/appcustom/` to add it to the userland image.

Use `<createos/uikit.h>` for the current terminal UI:

```c
#include <createos/uikit.h>

int main(void)
{
    createos_ui_t ui;
    if (createos_ui_init(&ui) < 0)
        return 1;
    if (createos_ui_clear() < 0 ||
        createos_ui_write_centered(&ui, ui.rows / 2,
                                   "Hello, CreateOS", 0x0F) < 0) {
        createos_ui_shutdown();
        return 1;
    }
    createos_ui_read_key();
    return createos_ui_shutdown() < 0;
}
```

The UI Kit writes to the console; it is not a desktop windowing API. Read
[application development](./app-development.md) for library, syscall, build,
and installation details.

## Learn Lua

Lua is the guest's interpreted scripting language. Start with a script such as
`hello.lua`:

```lua
print("Hello from CreateOS")
for count = 1, 3 do
    print("step", count)
end
```

Run it with `lua hello.lua`. Lua includes its base and standard libraries,
with native C modules disabled. Its OS and I/O functions are not a complete
POSIX host interface; in particular, `os.execute` and `io.popen` do not launch
programs. See [userland runtime details](../userland/userland.md).

## Downloading another language or library

There is no general-purpose "download a language" command in the guest. A
compiler/runtime must be ported to the CreateOS ABI, compiled for x86-64
freestanding execution, linked against supported syscalls/libraries, and
tested in QEMU before it can be advertised as a guest language.

When developing a port, add a source dependency only with a reviewed upstream
release or commit pinned in the build scripts, verify its published
cryptographic digest where available, keep downloaded source under ignored
`build/` storage, and document the exact guest features and missing
capabilities. Do not download a moving branch, bulk-copy unrelated language
implementations into the SDK, or treat a successful host build as proof that
guest execution works.
