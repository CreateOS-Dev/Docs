# C development on CreateOS

This chapter is a practical C guide for the toolchain, application ABI, and
libraries in this source tree. It teaches the parts needed to build programs
for CreateOS, then points out where the guest environment differs from a
desktop operating system. It is not a copy of the ISO C standard or of a
compiler manual: those documents describe a much larger environment than
this project provides.

The supported application path is:

1. Write C source (`.c`) and, where needed, x86-64 GAS source (`.S`).
2. Build with the pinned `x86_64-elf` cross-toolchain and `sdk/app.mk`.
3. Link a static ELF executable with the `.APPL` suffix.
4. Put that file in the image and execute it as a guest program.

The guest also includes TinyCC for small C programs. The compiler itself
does not make every host library or POSIX facility available.

## Contents

- [The executable model](#the-executable-model)
- [Your first SDK application](#your-first-sdk-application)
- [How the build finds source files](#how-the-build-finds-source-files)
- [C fundamentals](#c-fundamentals)
- [Functions and program structure](#functions-and-program-structure)
- [Arrays, pointers, and strings](#arrays-pointers-and-strings)
- [Structures and enums](#structures-and-enums)
- [Memory ownership](#memory-ownership)
- [Input and output](#input-and-output)
- [Files and paths](#files-and-paths)
- [Terminal applications](#terminal-applications)
- [Network helpers](#network-helpers)
- [Errors and diagnostics](#errors-and-diagnostics)
- [Header and library inventory](#header-and-library-inventory)
- [Working inside the guest with TinyCC](#working-inside-the-guest-with-tinycc)
- [Common mistakes](#common-mistakes)
- [Practice projects](#practice-projects)

## The executable model

An `.APPL` is native x86-64 ELF code. It is not a script, archive, or
sandboxed package. The suffix is used by the userland and app launcher to
recognize an application; it does not change the instruction set or make
untrusted code safe.

The SDK links the application's object files with CreateOS startup code and
static libraries. Its startup entry receives the process arguments, calls
`main`, and exits with the integer returned by `main`. Write the usual form:

```c
int main(void)
{
    return 0;
}
```

The current startup object also supports the conventional argument form:

```c
int main(int argc, char **argv)
{
    (void)argc;
    (void)argv;
    return 0;
}
```

The entry symbol is supplied by the SDK. Do not define `_start` in a normal
app. An app is statically linked; CreateOS does not provide a general dynamic
linker or a host-compatible POSIX environment.

An `.APPL` runs with the privileges of its userland execution path. It is
not isolated from the filesystem or kernel APIs by a package sandbox.
Validate input, check every result, and only execute programs you trust.

## Your first SDK application

From the repository root, build the cross-toolchain once:

```sh
make toolchain
```

The `hello` project in `sdk/examples/hello/` is the smallest sample. Its
complete source is:

```c
#include <createos/uikit.h>

int main(void)
{
    createos_ui_t ui;

    if (createos_ui_init(&ui) < 0)
        return 1;
    if (createos_ui_clear() < 0 ||
        createos_ui_write_centered(&ui, ui.rows / 2,
                                   "Hello from CreateOS!", 0x0F) < 0) {
        (void)createos_ui_shutdown();
        return 1;
    }
    (void)createos_ui_read_key();
    if (createos_ui_shutdown() < 0)
        return 1;
    return 0;
}
```

Build the example and then put the result in an image:

```sh
./sdk/scripts/build-example.sh hello
make userapps
make disk-image
make
make run
```

`make disk-image` creates a fresh disk from staged files. It does not merge
the staged root filesystem into an existing disk. Back up the old
`build/disk.img` first if it contains data you want to keep.

## Make a project

The SDK expects a project Makefile and C sources. A simple project layout is:

```text
notes/
├── Makefile
└── src/
    ├── main.c
    └── display/
        └── screen.c
```

Use a Makefile like this:

```make
APP_NAME := notes
APP_VERSION := 0.1.0
APP_SRC_DIR := $(CURDIR)/src
SDK_ROOT ?= /path/to/CreateOS/sdk

include $(SDK_ROOT)/app.mk
```

The SDK recursively discovers `.c` and uppercase `.S` files under
`APP_SRC_DIR`. Set `APP_NAME`, `APP_VERSION`, and `APP_SRC_DIR` before
including `app.mk`. The name and version are also passed to C as the string
macros `CREATEOS_APP_NAME` and `CREATEOS_APP_VERSION`.

Build with paths to the checkout's SDK and toolchain:

```sh
make -C /path/to/notes \
  SDK_ROOT="$PWD/sdk" \
  TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf"
```

The output is stored under `build/`; `BUILD_DIR` can override the default.
For a project with no `src/` directory, set `APP_SRC` to one C source file.
Use `APP_OBJECTS` only for object files that already match the target ABI.

### Install in the image

The SDK's `install` target copies the built app into the custom-app staging
directory:

```sh
make -C /path/to/notes install \
  SDK_ROOT="$PWD/sdk" \
  TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf" \
  CUSTOM_APP_DIR="$PWD/rootfs_maker/root/appcustom"
```

Then run `make userapps` to stage the userland. Rebuild `build/disk.img` only
after protecting any data on it. Apps in the custom directory are installed
under `/app/custom/` and can be launched from Explorer.

## How the build finds source files

The app build has three broad stages:

1. **Compile:** each discovered C or assembly source becomes an object file
   below the app's build directory.
2. **Link:** the objects are combined with startup, user, UI, network, and
   HTTP static libraries that the app references.
3. **Install (optional):** `make install` copies the `.APPL` into
   `CUSTOM_APP_DIR`.

The common Make variables are:

| Variable | Purpose |
| --- | --- |
| `APP_NAME` | Output program name and the app-name macro. |
| `APP_VERSION` | App-version macro; defaults to `0.1.0`. |
| `APP_DIR` | Project directory; defaults to the current Make directory. |
| `APP_SRC_DIR` | Root searched recursively for `.c` and `.S`; defaults to `src/`. |
| `APP_SRC` | Single C source fallback when no recursive source is found. |
| `APP_OBJECTS` | Additional target object files to link. |
| `APP_CFLAGS` | Extra C compiler options. |
| `APP_LDFLAGS` | Extra linker options. |
| `APP_MAIN_DEPS` | Extra prerequisites for app source objects. |
| `BUILD_DIR` | Generated outputs for this app. |
| `SDK_ROOT` | SDK location containing `app.mk` and `include/`. |
| `TOOLCHAIN_PREFIX` | Prefix containing the target compiler and linker. |
| `CUSTOM_APP_DIR` | Destination for `make install`. |

Do not set `CC` or `LD` to your host's native compiler. The project Makefile
chooses the cross compiler from `TOOLCHAIN_PREFIX`. A host x86-64 ELF binary
may look similar to a guest executable and still use the wrong libraries,
startup objects, or syscall ABI.

## C fundamentals

The SDK accepts ordinary C source compiled for its target. The language
examples below use standard C constructs; the available headers and runtime
functions are intentionally smaller than on a desktop distribution.

### Values and types

Use a type that describes the value and its range:

```c
#include <stdint.h>

int attempts = 0;
unsigned int rows = 25;
uint32_t screen_width = 80;
char answer = 'y';
const char *title = "CreateOS";
```

`int` is suitable for ordinary return values and small counts. Fixed-width
integer types such as `uint32_t` make binary layouts and syscall structures
clear. Include `<stdint.h>` for those types and `<stddef.h>` for `size_t`.
Do not assume that an `int` can represent every file size or buffer length.

`const` means that code through that expression must not modify the pointed
to object. It does not make the object globally immutable if another writable
pointer still refers to it.

### Expressions and comparisons

An assignment stores a value:

```c
attempts = attempts + 1;
attempts++;
```

Comparisons produce true or false:

```c
if (attempts >= 3) {
    puts("Try again later.");
}
```

Use `==` for equality and `=` for assignment. This common typo compiles:

```c
if (attempts = 3) {
    /* This assigns 3 and the condition is true. */
}
```

Use parentheses when precedence is not obvious. Split complicated
conditions into named boolean variables so that boundary cases are readable.

### Selection and repetition

```c
if (width == 0) {
    return -1;
} else if (width < 20) {
    puts("Small terminal");
} else {
    puts("Wide terminal");
}
```

A counted loop:

```c
for (uint32_t row = 0; row < ui.rows; row++) {
    if (createos_ui_fill_row(&ui, row, ' ', 0x07) < 0)
        return 1;
}
```

A condition-controlled loop:

```c
int key;
do {
    key = createos_ui_read_key();
} while (key != 'q' && key != 'Q');
```

Choose loop bounds carefully. If an array has `count` entries, valid indexes
are from zero through `count - 1`; the condition should usually be
`index < count`.

### Functions and return values

Declare a function before calling it, usually through a header:

```c
static int is_valid_width(uint32_t width)
{
    return width > 0 && width <= 4096;
}
```

`static` on a function limits its name to the current source file. This is
useful for implementation details. A header should declare functions that
other source files call; do not put a non-`static` function definition in a
header that multiple files include.

Use return values to report failure. If a helper returns `0` on success and
negative on failure, propagate the error rather than continuing with
uninitialized data:

```c
if (createos_ui_refresh_dimensions(&ui) < 0) {
    puts("Unable to read terminal dimensions.");
    return 1;
}
```

The exact convention belongs to each API; do not assume every function uses
the same convention.

## Functions and program structure

Keep `main` responsible for the program's sequence, not every implementation
detail. A small app can be split into:

- `main.c`: initialization, main loop, and exit status.
- `input.c`: key interpretation and editing.
- `screen.c`: redraw helpers.
- `app.h`: structures and prototypes shared between source files.

Example header:

```c
#ifndef NOTES_APP_H
#define NOTES_APP_H

int notes_render(void);

#endif
```

Include guards prevent a header from being processed repeatedly by one
compilation unit. Include the declarations a source uses directly rather
than relying on another header to include them accidentally.

Use descriptive names that explain units and ownership. `timeout_ms` is
clearer than `timeout`; `buffer_capacity` is clearer than `size` when there
are multiple sizes in scope.

### Scope and lifetime

A local variable exists while its function is executing. A static local
persists for the life of the process:

```c
static unsigned long next_id(void)
{
    static unsigned long current;
    return ++current;
}
```

Avoid returning the address of a local variable. Its lifetime ends when the
function returns:

```c
int *wrong(void)
{
    int value = 7;
    return &value;
}
```

Return a value, let the caller provide storage, or allocate an object whose
ownership is clearly transferred.

### Translation units

Each `.c` file is compiled independently. A declaration in one source file
does not automatically make a function visible in another. Put shared
declarations in a header and include it from both callers and definitions.
The linker then connects the compiled references.

Keep headers self-contained: if a declaration uses `uint32_t`, the header
should include `<stdint.h>` itself. This prevents fragile dependencies on
include order.

## Arrays, pointers, and strings

An array is a fixed-size sequence:

```c
char name[32];
```

Its capacity is 32 bytes. A C string needs one byte for its terminating
zero, so it can hold at most 31 ordinary one-byte characters. Always reserve
space for that terminator.

A pointer contains an address:

```c
char *cursor = name;
```

Before dereferencing a pointer, establish that it refers to a valid object.
Before indexing, establish that the index is in range. C does not add a
runtime bounds check.

### Bounded text

Prefer a capacity-aware formatter for constructed strings:

```c
#include <stdio.h>

char message[64];
int count = snprintf(message, sizeof(message), "items: %d", 3);
if (count < 0 || (size_t)count >= sizeof(message))
    return 1;
```

The CreateOS formatter's currently implemented conversions are `%s`, `%d`,
`%i`, `%c`, `%x`, `%X`, and `%%`. Do not assume a desktop libc's entire
`printf` format language is present. For portable app messages, use those
conversions and verify formatting behavior in the current user library.

`strcpy` and `strcat` do not know the destination capacity. Only use them
when you have already proved that the destination is large enough. For a
fixed buffer, track its current length and remaining capacity explicitly.

### Byte buffers and text

Text is not always a C string. A file read can return arbitrary bytes,
including embedded zero bytes. Keep the returned byte count and do not call
`strlen` on arbitrary file contents:

```c
#include <unistd.h>

unsigned char bytes[128];
ssize_t received = read(fd, bytes, sizeof(bytes));
if (received < 0) {
    /* Inspect errno or report an I/O error. */
    return 1;
}
/* Process exactly received bytes, not strlen((char *)bytes). */
```

Append a terminator only when the data is text, the buffer has spare space,
and the bytes have been validated for your intended use.

### Memory operations

`memcpy` requires non-overlapping ranges. Use `memmove` when source and
destination may overlap. `memcmp` compares bytes; it does not compare
human-language strings according to locale rules.

Check that size calculations cannot overflow before allocating or indexing.
For example, if computing `count * sizeof(item)`, verify that `count` is no
greater than `SIZE_MAX / sizeof(item)` first.

## Structures and enums

Use structures to keep related values together:

```c
typedef struct {
    uint32_t width;
    uint32_t height;
    int active;
} display_state_t;
```

An enum makes state names easier to review:

```c
typedef enum {
    VIEW_HOME,
    VIEW_FILES,
    VIEW_ABOUT
} view_t;
```

When passing pointers to structures through a syscall or shared ABI, use the
exact definition from the project header. Field order, field width, padding,
and alignment matter. Do not recreate a kernel structure from memory or
change a shared structure in only one component.

Do not write a C structure directly to disk as a persistent file format
unless the format explicitly fixes its byte order and layout. Compiler
padding and ABI details can change the bytes. Encode fields explicitly for
formats that need long-term compatibility.

## Memory ownership

The user library supplies `malloc`, `calloc`, `realloc`, and `free`.
Allocation can fail:

```c
char *buffer = malloc(256);
if (!buffer) {
    puts("Not enough memory.");
    return 1;
}

/* Use buffer. */
free(buffer);
```

For every successful allocation, decide who owns it and where it is freed.
Free it on normal and error paths. Do not:

- free the same pointer twice;
- use a pointer after freeing it;
- pass a pointer to `free` that did not come from an allocator;
- lose the only pointer to an allocation before freeing it;
- assume `realloc` leaves the original allocation untouched on failure
  without following the allocator's documented contract.

The current user allocator uses syscall-backed memory regions; it is not a
kernel heap API. App code should use the user library rather than calling
kernel allocation internals.

## Input and output

The guest C library is small. For line-oriented output:

```c
#include <stdio.h>

puts("Ready.");
printf("attempt %d\n", 2);
```

`puts` writes a string and newline. `putchar` writes one character.
`printf` is convenient for diagnostics, but only use conversions known to
be implemented by this userland.

For terminal programs requiring coordinates, colors, keyboard input, or
mouse position, use `<createos/uikit.h>` rather than hand-writing escape
sequences. The [UI Kit guide](uikit.md) documents its return values and
limits.

Standard output calls are not a general terminal-control protocol. Check
return values when the program must know whether a write succeeded.

## Files and paths

The available POSIX-like subset is implemented in
`rootfs_maker/lib/posix.c`. Include the corresponding project header and
check its declaration before using a function. For example:

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int copy_prefix(const char *path)
{
    char buffer[128];
    int fd = open(path, O_RDONLY);
    if (fd < 0) {
        puts("open failed");
        return 1;
    }

    ssize_t count = read(fd, buffer, sizeof(buffer));
    if (count < 0) {
        (void)close(fd);
        puts("read failed");
        return 1;
    }

    if (count > 0 && write(1, buffer, (size_t)count) < 0) {
        (void)close(fd);
        puts("write failed");
        return 1;
    }
    if (close(fd) < 0) {
        puts("close failed");
        return 1;
    }
    return 0;
}
```

The descriptor values and filesystem semantics belong to CreateOS. The
headers provide `open`, `read`, `write`, `close`, `lseek`, `access`,
`chdir`, `unlink`, `mkdir`, `stat`, and related declarations, but this does
not mean every POSIX flag or edge case is implemented. Check
`rootfs_maker/lib/fcntl.h`, `unistd.h`, `sys/stat.h`, and their implementation
before relying on a flag.

Paths use the guest filesystem namespace. A relative path is interpreted
relative to the process's working directory. Permissions and the active
account apply. The host cannot treat `build/rootfs/` as the live guest disk;
the persistent disk is a CFS5 image.

### Errors and `errno`

The POSIX adapter usually returns `-1` and sets `errno` for failures. The
lower-level `sys_*` interface can use a different negative-result
convention. Do not mix the two conventions in one function without an
explicit conversion.

Capture the return value before using output buffers. On failure, do not
assume a buffer contains a complete result. Close descriptors on every path
after a successful `open`.

## Terminal applications

The UI Kit API is a small static library layered over the TTY syscall
interface. It offers cell-based text operations, not windows or controls.
The standard pattern is:

1. Call `createos_ui_init`.
2. Draw the initial screen.
3. Read keys and update application state.
4. Redraw changed content.
5. Call `createos_ui_shutdown` before returning.

Always check initialization. If it succeeds, arrange cleanup for every
later exit path. The [UI Kit reference](uikit.md) describes screen bounds,
colors, input, mouse-position packing, and error results.

Applications that manage their own cursor, mouse selection, or clipboard
should consult the declarations in `rootfs_maker/lib/user_syscall.h`.
These are CreateOS interfaces, not portable terminal APIs.

## Network helpers

The SDK has narrow APIs rather than BSD sockets:

- `<createos/network.h>` wraps the TCP listener, accept, read, write, and
  close operations available to the app.
- `<createos/http.h>` contains a bounded HTTP request parser.

The HTTP parser reports a complete, incomplete, or invalid request. It does
not open a connection, implement TLS, or provide a full HTTP server. Network
operations can fail due to missing interfaces, timeouts, permissions, or
remote behavior. Treat received lengths as bounds and do not parse past
them. See the API headers in `sdk/include/createos/` before selecting
buffer sizes or relying on a return convention.

## Header and library inventory

The installed guest headers come from `rootfs_maker/lib/`, the kernel's
public CreateOS headers, and TinyCC's small C headers. The SDK exposes its
own public headers under `sdk/include/createos/`.

| Header | Examples of declarations |
| --- | --- |
| `<stddef.h>` | `size_t`, `NULL`, `offsetof` |
| `<stdint.h>` | fixed-width integers |
| `<stdbool.h>` | `bool`, `true`, `false` |
| `<stdarg.h>` | variable argument lists |
| `<string.h>` | string and byte operations |
| `<stdlib.h>` | allocation, `atoi`, `rand`, `abort` |
| `<stdio.h>` | formatted and basic output, `rename`, `remove` |
| `<fcntl.h>` | `open` flags and declaration |
| `<unistd.h>` | descriptor I/O and process/path helpers |
| `<sys/stat.h>` | stat structures and metadata operations |
| `<sys/wait.h>` | `wait` |
| `<dirent.h>` | directory-entry interface |
| `<ctype.h>` | character classification |
| `<time.h>` | time structures and formatting |
| `<createos/uikit.h>` | text terminal helper API |
| `<createos/network.h>` | CreateOS TCP helpers |
| `<createos/http.h>` | bounded HTTP request parsing |

This inventory is a navigation aid, not a promise of full ISO C or POSIX
coverage. Some headers are compatibility shims. Consult the actual header
and implementation when a function's exact behavior matters.

### Formatting subset

The current formatter implements `%s`, `%d`, `%i`, `%c`, `%x`, `%X`, and
`%%`. It is not safe to assume support for `%f`, width/precision modifiers,
length modifiers, `%p`, or unsigned conversions merely because another
`printf` supports them. If output needs a format not listed here, convert it
explicitly or extend and test the runtime before depending on it.

## Working inside the guest with TinyCC

TinyCC is useful for small experiments and utilities. A simple guest build
looks like this:

```text
tcc -o /home/root/hello hello.c
chmod u+x /home/root/hello
/home/root/hello
```

The program must use headers and static libraries installed in the guest.
Dynamic linking and TinyCC's `-run` feature are not provided. A host-built
binary is not a substitute for a CreateOS-targeted binary.

For repeatable development, source discovery, app metadata, assembly
objects, SDK libraries, and the `.APPL` suffix, prefer the host SDK guide.
TinyCC is a guest tool, not the SDK's cross compiler.

## Common mistakes

### Using the host compiler by accident

`cc hello.c -o hello` targets the host. It may produce an executable that
cannot run in the guest. Use `make toolchain` and the SDK.

### Assuming libc is complete

The build can succeed while a call or format expected from a desktop libc
does not exist. Check the installed project headers and source rather than
guessing from a familiar function name.

### Ignoring lengths

Every buffer has a capacity. Keep the number of bytes returned by `read`,
bound every copy, and reserve a terminator when converting bytes to a C
string.

### Forgetting cleanup

Close opened descriptors, free owned allocations, and leave full-screen UI
mode before returning, including on failures.

### Treating application files as safe packages

An `.APPL` is executable code. The suffix does not provide isolation or
permission review. Build from sources you trust and inspect any app before
installing it in a disk image.

## Practice projects

Work through these in order. Build each one with the SDK and run it in QEMU.

1. **Hello screen:** center a title, wait for a key, and restore the terminal.
2. **Argument viewer:** print `argc` and each `argv` element, one per line.
3. **Bounded counter:** parse one small integer argument and reject malformed
   or out-of-range input.
4. **Text file preview:** open a path, read a fixed-size byte buffer, and
   display exactly the bytes read.
5. **Directory inventory:** use the directory API to list one path without
   writing beyond its buffer.
6. **Two-screen app:** add a small state enum and draw a different page for
   each key.
7. **Assembly helper:** implement a small leaf function in `.S` and call it
   from C; see [the assembly guide](assembly.md).
8. **Lua companion:** implement the same small text-processing task in Lua
   and compare the runtime constraints in [the Lua guide](lua.md).

For every project, test empty input, a boundary-sized buffer, a missing file,
and a failing syscall. A successful build checks syntax and linking; it
does not prove that error paths or terminal restoration work.
