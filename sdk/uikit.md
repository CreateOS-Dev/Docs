# CreateOS UI Kit

The UI Kit is the SDK's small C library for text-mode applications. It
draws characters into the active CreateOS terminal and reads terminal
input. It does not create graphical windows, buttons, menus, or a desktop
compositor.

The public header is `sdk/include/createos/uikit.h`; the implementation is
`sdk/src/uikit.c`. The SDK builds `libcreateos-uikit.a` and links it into
applications automatically.

This guide is organized as a working manual: first build the sample, then
learn the screen model, lifecycle, drawing calls, input, error handling,
and patterns for small terminal interfaces.

## Contents

- [Build the examples](#build-the-examples)
- [The terminal screen model](#the-terminal-screen-model)
- [Start and restore the UI](#start-and-restore-the-ui)
- [Drawing text](#drawing-text)
- [Colors](#colors)
- [Reading keys](#reading-keys)
- [Reading the mouse position](#reading-the-mouse-position)
- [Handling dimensions](#handling-dimensions)
- [Errors and cleanup](#errors-and-cleanup)
- [A complete menu example](#a-complete-menu-example)
- [Designing a redraw loop](#designing-a-redraw-loop)
- [What the UI Kit does not provide](#what-the-ui-kit-does-not-provide)
- [Troubleshooting](#troubleshooting)

## Build the examples

From the complete source tree, build the pinned cross-toolchain and compile
the UI demo:

```sh
make toolchain
./sdk/scripts/build-example.sh uikit-demo
```

The output is in the SDK build directory under `build/`. To include the
current app in the OS image and run it:

```sh
make userapps
make disk-image
make
make run
```

The image step recreates the disk; back it up first if it contains user
files you need to retain. The sample source is
`sdk/examples/uikit-demo/main.c`. The smaller `hello` example also shows
initialization, centered text, key input, and shutdown.

## The terminal screen model

The UI Kit talks to the terminal active for the application. Its dimensions
are expressed in character cells:

- `columns` is the number of cells across.
- `rows` is the number of cells down.
- column 0, row 0 is the top-left cell.
- column `columns - 1`, row `rows - 1` is the bottom-right cell.

The structure passed to UI Kit calls is:

```c
typedef struct {
    uint32_t columns;
    uint32_t rows;
} createos_ui_t;
```

Include `<createos/uikit.h>` to use the structure and functions. The header
includes `<stdint.h>` for its integer types.

The UI Kit draws characters in terminal cells. It does not measure pixel
fonts or provide layout containers. A terminal cell count is not a promise
about display pixel dimensions.

## Start and restore the UI

Initialize before using the screen dimensions or drawing:

```c
#include <createos/uikit.h>

int main(void)
{
    createos_ui_t ui;

    if (createos_ui_init(&ui) < 0)
        return 1;
    /* Draw and read input here. */
    if (createos_ui_shutdown() < 0)
        return 1;
    return 0;
}
```

`createos_ui_init` refreshes the terminal dimensions and begins full-screen
mode. If it cannot query the active terminal name, initialization fails.
When the name is known but no valid size is found among the reported
terminals, the helper uses a fallback size of 80 columns by 25 rows.

After successful initialization, call `createos_ui_shutdown` before every
normal return. A helper that can fail should return to one cleanup path:

```c
int main(void)
{
    createos_ui_t ui;
    int result = 1;

    if (createos_ui_init(&ui) < 0)
        return 1;

    if (createos_ui_clear() < 0)
        goto done;
    if (createos_ui_write_centered(&ui, 2, "Ready", 0x0F) < 0)
        goto done;

    result = 0;
done:
    if (createos_ui_shutdown() < 0)
        result = 1;
    return result;
}
```

This example uses `goto` only to centralize cleanup; it does not jump into a
scope or bypass resource initialization. For a larger app, use a small
cleanup function or a clearly structured state machine.

## API reference

| Function | Parameters and behavior | Return |
| --- | --- | --- |
| `createos_ui_init(ui)` | Requires a writable, non-null `createos_ui_t`. Refreshes dimensions and begins full-screen mode. | `0` on success; negative on failure. |
| `createos_ui_refresh_dimensions(ui)` | Re-queries the active TTY and updates `columns` and `rows`. | `0` on success; `-1` for a null pointer or failed TTY-name query. |
| `createos_ui_clear()` | Clears the current terminal using the user console syscall. | The result of `sys_clear`. |
| `createos_ui_write_at(ui, column, row, text, color)` | Checks pointers and coordinate bounds, moves the cursor, and writes `text`. | `-1` on invalid arguments or cursor failure; otherwise the text syscall result. |
| `createos_ui_write_centered(ui, row, text, color)` | Clips the string length to the terminal width, centers it, then writes characters. | `-1` on invalid arguments or a terminal write failure; otherwise `0`. |
| `createos_ui_fill_row(ui, row, character, color)` | Writes one character to each column of the selected row. | `-1` if invalid or too wide for its safety limit; otherwise `0`. |
| `createos_ui_read_key()` | Reads a key with the blocking `sys_getc` interface. | Key value converted to an unsigned byte and returned as `int`. |
| `createos_ui_mouse_position()` | Reads the current terminal pointer cell position. | Packed position from the TTY syscall. |
| `createos_ui_shutdown()` | Ends full-screen terminal mode. | The result of `sys_tty_fullscreen_end`. |

The function implementations are short; when behavior matters, compare this
table with the current source in `sdk/src/uikit.c` and the syscall
declarations in `rootfs_maker/lib/user_syscall.h`.

## Drawing text

### Clear the screen

```c
if (createos_ui_clear() < 0) {
    /* Report or handle the terminal failure. */
}
```

Clear once before an initial full-screen draw, or when changing to a screen
that should replace everything. Clearing on every keypress can cause
flicker; update only the cells that changed when practical.

### Write at a cell

```c
if (createos_ui_write_at(&ui, 2, 4, "Settings", 0x0F) < 0)
    return 1;
```

Coordinates are zero-based. The function checks that the requested starting
column and row are inside the dimensions stored in `ui`. It does not clamp
an invalid coordinate.

The helper does not truncate `text` to the remaining cells on the row.
Long text may continue according to the terminal's normal output behavior.
If the string must fit, compute its visible length and clip it before
calling `write_at`.

```c
static size_t clip_length(const char *text, size_t maximum)
{
    size_t length = 0;
    while (text[length] && length < maximum)
        length++;
    return length;
}
```

To turn a clipped length into a C string, copy into a destination buffer
with one byte reserved for the terminating zero. Do not modify a string
literal to add that terminator.

### Center text

```c
if (createos_ui_write_centered(&ui, 2, "CreateOS Settings", 0x0B) < 0)
    return 1;
```

The helper calculates the centered column from the string's byte length,
limited to the terminal width. It writes each character in sequence. It
does not provide Unicode display-cell measurement; multi-byte UTF-8 text
may not have a length equal to the number of terminal cells it occupies.
For predictable alignment, use ASCII labels or implement a project-specific
width calculation.

### Fill a row

```c
if (createos_ui_fill_row(&ui, 4, '-', 0x09) < 0)
    return 1;
```

The row must be less than `ui.rows`. The current implementation refuses
dimensions wider than 4096 columns as a safety limit, then writes one
character per cell.

## Colors

The color argument is passed to the console output operation. The
framebuffer console maps its low four bits to this VGA-style palette:

| Value | Foreground |
| --- | --- |
| `0x0` | black |
| `0x1` | blue |
| `0x2` | green |
| `0x3` | cyan |
| `0x4` | red |
| `0x5` | magenta |
| `0x6` | brown |
| `0x7` | light gray |
| `0x8` | dark gray |
| `0x9` | bright blue |
| `0xA` | bright green |
| `0xB` | bright cyan |
| `0xC` | bright red |
| `0xD` | bright magenta |
| `0xE` | yellow |
| `0xF` | white |

Examples in the tree use `0x0F` for white, `0x0A` for bright green, `0x09`
for bright blue, and `0x07` for light gray. The framebuffer implementation
uses the low nibble as foreground color; do not assume the UI Kit documents
a portable background-color encoding.

## Reading keys

The UI Kit exposes one key read at a time:

```c
int key = createos_ui_read_key();
if (key == 'q' || key == 'Q') {
    /* Request application exit. */
}
```

The function uses the blocking `sys_getc` syscall. It does not have a
timeout argument, does not return a whole line, and converts the returned
character to an unsigned byte; it does not provide a separate negative
error result. A line editor must collect keys into a bounded buffer and
handle backspace, Enter, and overflow itself.

ASCII characters are returned as their byte values. Special-key constants
such as `KEY_LEFT`, `KEY_RIGHT`, and `KEY_DELETE` are declared in
`rootfs_maker/lib/user_syscall.h`, not in the public UI Kit header. SDK
applications can include that userland API header when they need those
CreateOS-specific values:

```c
#include "user_syscall.h"
#include <createos/uikit.h>
```

Because these values are part of CreateOS's terminal ABI, follow the
declarations from the same checkout as the kernel.

The key read is blocking. A loop that calls it waits for user input and
does not periodically inspect other state. Use a lower-level poll interface
from `user_syscall.h` only when your app needs non-blocking input, and check
its return convention in the implementation.

## Reading the mouse position

`createos_ui_mouse_position()` returns the current pointer cell position
packed into one integer:

```c
int packed = createos_ui_mouse_position();
if (packed < 0) {
    /* The syscall reported an error. */
} else {
    uint32_t x = (uint32_t)packed & 0xFFFFu;
    uint32_t y = ((uint32_t)packed >> 16) & 0xFFFFu;
}
```

The kernel packs x in the low 16 bits and y in the high 16 bits. The helper
does not return mouse buttons, movement history, or a stream of events. Do
not interpret a single position query as a click notification.

For event-driven mouse input, an app must use the lower-level terminal
input API and handle its documented key values. The UI Kit itself does not
provide a mouse event loop or widget hit-testing.

## Handling dimensions

`ui.columns` and `ui.rows` are the dimensions measured at initialization or
at the last explicit refresh. If a program changes terminal or expects a
resize:

```c
if (createos_ui_refresh_dimensions(&ui) < 0)
    return 1;
```

Then clamp layout to the new values. Avoid unsigned underflow when a
terminal is smaller than your layout:

```c
if (ui.columns < 20 || ui.rows < 5) {
    (void)createos_ui_clear();
    (void)createos_ui_write_at(&ui, 0, 0, "Terminal too small", 0x0C);
} else {
    /* Draw the full interface. */
}
```

The helper does not automatically receive a resize notification and
redraw the screen. Your app must decide when to refresh and then repaint.
If dimensions can be zero in an error path, check them before subtracting
one or computing `rows - 2`.

## A complete menu example

This small menu illustrates initialization, clearing, drawing, a key loop,
and one cleanup path. It uses a small static selection rather than
allocating memory:

```c
#include <createos/uikit.h>

static int draw_menu(const createos_ui_t *ui, unsigned int selected)
{
    static const char *items[] = {
        "Files",
        "Settings",
        "About",
        "Quit"
    };
    const unsigned int count =
        (unsigned int)(sizeof(items) / sizeof(items[0]));

    if (createos_ui_clear() < 0)
        return -1;
    if (createos_ui_write_centered(ui, 1, "CreateOS menu", 0x0B) < 0)
        return -1;

    for (unsigned int i = 0; i < count; i++) {
        unsigned int row = 4 + i;
        int color = i == selected ? 0x0F : 0x07;
        if (createos_ui_write_centered(ui, row, items[i], color) < 0)
            return -1;
    }
    return 0;
}

int main(void)
{
    createos_ui_t ui;
    unsigned int selected = 0;
    int result = 1;

    if (createos_ui_init(&ui) < 0)
        return 1;
    if (ui.rows < 8 || ui.columns < 24)
        goto done;
    if (draw_menu(&ui, selected) < 0)
        goto done;

    for (;;) {
        int key = createos_ui_read_key();
        if (key == 'q' || key == 'Q')
            break;
        if (key == 'j' && selected < 3)
            selected++;
        if (key == 'k' && selected > 0)
            selected--;
        if (draw_menu(&ui, selected) < 0)
            goto done;
    }
    result = 0;

done:
    if (createos_ui_shutdown() < 0)
        result = 1;
    return result;
}
```

This example uses `j` and `k`, so it works without importing special-key
constants. A production menu should display its key instructions, validate
dimensions before drawing, and decide what each selected item does.

## Designing a redraw loop

Keep UI state separate from drawing:

```c
typedef struct {
    unsigned int selected;
    int dirty;
    int running;
} menu_state_t;
```

A useful loop has three distinct phases:

1. Read one input event.
2. Update state and mark which region needs repainting.
3. Draw only when the state changed.

This structure avoids putting selection rules inside drawing code. It also
makes the input behavior testable without a full framebuffer session.

For a short static app, redrawing the entire screen on every key is
acceptable. For a larger editor or explorer, redraw only the changed row or
region to avoid visual flicker and unnecessary syscalls.

Do not build strings into a small local array with unbounded formatting.
Use `snprintf`, check its result, and keep the formatter subset in
CreateOS's C library in mind. See the [C programming guide](c.md).

## What the UI Kit does not provide

The public API does not provide:

- windows, panels, widgets, or a window manager;
- text input fields or line editing;
- a non-blocking event queue;
- mouse button state through the position helper;
- scrolling regions or a retained screen buffer;
- Unicode grapheme measurement or font layout;
- automatic resize notification and redraw;
- a general graphics API.

Use the CreateOS terminal syscalls for lower-level behavior, or implement
small app-specific helpers over the UI Kit. Avoid describing those helpers
as part of the public library unless they are added to its header and
implementation.

## Troubleshooting

### The app exits before drawing

Check the return from `createos_ui_init`. It can fail before full-screen
mode begins. Report an error through the ordinary console path if the
terminal is available.

### Text is not centered as expected

The helper uses string length in bytes, not Unicode display width. Try
ASCII text first and check the actual row and `ui.columns`.

### Text continues onto the next line

`createos_ui_write_at` checks the start coordinate but does not clip the
whole string to the remaining row. Clip it before writing if wrapping would
be wrong.

### The mouse value looks negative

The API returns a signed `int`, although the position is packed from two
16-bit coordinates. Interpret successful results as `uint32_t` before
extracting x and y; verify the syscall's error behavior before treating a
negative result as coordinates.

### The screen remains in full-screen mode

Make sure every path after successful initialization reaches
`createos_ui_shutdown`, including errors during drawing or input.

For project layout and install instructions, read
[application development](app-development.md). For examples of real apps
using the API, inspect the source in `sdk/examples/uikit-demo/` and
`rootfs_maker/root/app/settings/`.
