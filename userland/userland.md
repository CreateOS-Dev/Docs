# Userland, shell, and editor

## Build and filesystem layout

User programs are built by `rootfs_maker/Makefile` using the selected
`x86_64-elf` compiler and linker. Shell, login, init, and command programs are
staged into `/bin` or `/sbin`; standalone applications use the `.APPL` format
and are staged in `/app/system/bin`. A `.link` file under `/app/system/` is a
text shortcut to an application target; see the existing
`rootfs_maker/root/app/system/` examples.
Each account gets `~/Applications` with `.APPL` shortcuts for Explorer,
TextEdit, and Settings. `open` lists these personal shortcuts and resolves
them to the installed applications in `/app/system/bin`. Userland initializes
the home configuration, keyboard-layout directory, and application shortcuts
after account creation and at login; the kernel stores account credentials and
enforces filesystem access, but does not populate those user files. Non-root
accounts use distinct filesystem UIDs derived from their account-table slots.
New account names must contain only ASCII letters (`A-Z`, `a-z`); digits,
spaces, punctuation, and other characters are rejected by FirstBoot, `adduser`,
and the kernel account-creation API. Existing account records remain readable.
On successful login, the kernel changes ownership of the account's existing
CFS home tree and fails login if that bounded ownership scan cannot complete.
New files and directories below a home receive that account's UID. Keep account
record ordering stable: changing it changes the derived UID, and the home tree
is reassigned the next time that account logs in.

The shell resolves command names in `/bin` and `/usr/bin`, including the
current `.bin` command filename convention. Lua is available as `lua` through
`/bin/lua.bin`; that command immediately replaces itself with the installed
`/app/system/bin/lua.APPL` interpreter. The same application is discoverable
through the `/app/system/lua.link` shortcut. The shell has line editing,
command history, and path/application completion; completion suggestions are
kept below the current prompt and are hidden when no screen rows remain, so
Tab cycling never scrolls the console. Its parser is currently whitespace
based; do not expect POSIX quoting, pipelines, redirection, expansion, job
control, or a complete shell language.
Pinned TinyCC source is vendored under `kernel/lib/third_party/tinycc`, but is
compiled only into the guest application `/app/system/bin/tcc.APPL`; the
`/bin/tcc.bin` command wrapper forwards its arguments to that application.
The compiler uses CreateOS headers in `/usr/include` and its static startup,
libc, and `libtcc1` archives in `/usr/lib/tcc`. The kernel provides filesystem,
process, memory, and time services through its existing syscalls; TinyCC and
compiled applications remain unprivileged ring-3 programs. The compiler
supports static CreateOS executables, not dynamic linking or TinyCC's `-run`
execution mode. Compile with `tcc -o /home/root/hello hello.c`, then launch the
result by its path from the shell.

DHCP startup is handled by a compiled userland hook in init and first-boot
setup, controlled by `/etc/network.conf` (`autostart=1` or `autostart=0`).
This is not a shell script: the guest shell does not yet interpret scripts.
`curl URL` streams the response body to the terminal, so it is recorded in the
console scrollback and can be reviewed after the request completes. HTTP
headers are skipped; chunked transfer framing and redirects are not decoded.
`curl -o PATH URL` accepts only a relative path below the current user's home,
validates the HTTP status and response framing, and holds the complete
response in application memory until the user confirms saving it. Existing
files require a second explicit overwrite confirmation. Declined or invalid
downloads are discarded without writing a file. The download limit is 4 KiB
including headers; chunked transfer encoding and larger files are rejected
rather than partially saved. HTTPS also requires a valid clock, hardware
entropy, and a trusted CA. This is a confirmation and storage boundary, not
malware scanning: saving a document does not prove that its contents are safe.

Lua 5.4.8 is installed as `/app/system/bin/lua.APPL`. Its base, coroutine,
debug, I/O, math, OS, package, string, table, and UTF-8 standard libraries are
opened. Lua files can be loaded from the current directory, the user's home,
and `/usr/share/lua/5.4`; native C modules are disabled. Native applications
can synchronously run a child image through `sys_run`, but Lua does not yet
connect `os.execute` or `io.popen` to that API, and CreateOS still has no
POSIX `fork`/`wait`. Lua calendar functions use UTC, and its freestanding math
functions do not provide system-`libm` precision for exceptionally large
trigonometric arguments. See [the Lua app notes](../rootfs_maker/root/app/lua/README.md)
for invocation and port details. The `luac` command compiles scripts to
bytecode with the separate `/app/system/bin/luac.APPL` application. This is
not a C compiler; CreateOS still has no guest `cc`.

## Network exposure

CreateOS provides DHCP, DNS resolution, ICMP echo, and outbound TCP
connections. System-privileged applications can now use a custom, admin-gated
TCP listen/accept API on one selected interface and port, with one accepted
stream at a time. This is not POSIX sockets or a general-purpose
packet-filtering firewall; device-specific and packet-level tests are still
required before claiming that inbound traffic is controlled on every adapter.
The default QEMU user-network configuration has no host port forwards.

The public app SDK exposes this limited interface in
`<createos/network.h>`/`libcreateos-network.a`. Nginx is not yet a usable guest
application: its upstream Unix/Linux build requires a guest libc/filesystem
runtime and event-polling interfaces that CreateOS does not provide. The TCP
listener API is only an initial prerequisite, not a port of Nginx or a
general-purpose server runtime.
The SDK also contains a bounded HTTP/1.0/1.1 request parser in
`<createos/http.h>`/`libcreateos-http.a`; it is a parsing utility, not an HTTP
server. The parser currently has no maintained automated test suite; successful
compilation alone does not validate malformed or partial requests.
The built-in `httpd.APPL` is a one-request diagnostic server on TCP port 8080,
serving only fixed responses for `/` and `/health`. It exercises the current
single-listener API and exits after one request; it is not Nginx or suitable
for untrusted/public network exposure. QEMU requires explicit host forwarding.

## Text editor

`textedit` is an interactive editor with a 4096-byte per-file limit, up to
eight open tabs, `Ctrl+O` to open another file, and `Ctrl+Tab` to switch tabs.
Use Shift+Arrow or Shift+Home/End, or drag with the left mouse button, to
select text; `Ctrl+C` copies the selection to the editor's in-memory clipboard
and `Ctrl+V` inserts it or replaces the current selection. The clipboard is
not persisted and is shared between tabs only for the lifetime of the TextEdit
process. The insertion caret blinks, and the style can be switched between a
vertical `|` and an underline `_` in Settings.
`Esc` returns to Explorer when opened there; unsaved tabs prompt to save all,
discard all, or cancel. It supports cursor movement and lightweight source
highlighting for common C tokens, strings, numbers, comments, and preprocessor
lines; CRLF and legacy CR line endings are normalized to LF when loaded. The
highlighter is not a full C parser. The screen shows the application,
active file, full path, saved state, and tab names. On serial or VGA text-only
consoles it uses the traditional 80x25 layout with 17 visible editor lines; it
is not a pixel-based desktop interface. Vertical cursor movement preserves the
logical column and chooses the nearest side when that column falls inside a tab.
TextEdit and the shell share a system clipboard: drag or use Shift+Left/Right to
select text, then Ctrl+C to copy, Ctrl+X to cut, and Ctrl+V to paste. In the
shell, Ctrl+C cancels the current command line when there is no selection.
In other foreground applications, Ctrl+C exits the application and returns to
the shell, including while the app waits for a TCP connection or data. TextEdit
keeps Ctrl+C for copying selected text.
TextEdit also provides a navigable file-tree sidebar with Ctrl+B (arrow keys
move, Enter opens a folder or file, and `..` goes to the parent directory).
Launching it without a path browses the current working directory; passing a
folder opens that folder as the workspace. Relative paths resolve from the
current directory/workspace. For `.lua` files, Ctrl+R executes the current
buffer in an embedded Lua runtime, including unsaved changes; any key returns
to the editor after execution. Python is not installed, and C/Makefile
execution is unavailable because the guest currently has no `make` or `cc`.
On a `.c` file, Ctrl+R checks for a Makefile and creates a CreateOS SDK
Makefile template if none exists, without replacing an existing one; it does
not pretend to compile the project.

First boot presents the language, keyboard, region, time zone, username, and
password choices as separate numbered steps. After account creation it displays
a status log for the account/home setup, saved preferences, and completion
marker. The completion marker is written only after every selected preference
has been saved; a failed settings write reports incomplete setup instead of
claiming success. Password characters are masked and are never included in the
log. After saving the completion marker, FirstBoot synchronizes the filesystem
and requests a system reboot; if synchronization or the reboot request fails,
it reports the failure and falls back to starting init.

## Keyboard layouts and cursor themes

`keymap` reports or selects the installed `us.lyt` and `fr-azerty.lyt`
keyboard layouts. `cursor` reports or selects the installed
`classic.srbmp`, `high-contrast.srbmp`, `inverted.srbmp`, `createos.srbmp`,
`blue-arrow.srbmp`, `red-arrow.srbmp`, `crosshair.srbmp`, `precision.srbmp`,
and `hand.srbmp` pointer themes. The OS also uses an I-beam text pointer
automatically in TextEdit, and shows a rotating busy indicator only while an
executable is loading. All pointer assets are bounded monochrome-mask sprites
with configurable foreground and outline colors; crosshair and precision use
their configured hotspots, the crosshair uses a cross shape, and hand/text
pointers use their natural hotspots.
The pointer switches automatically to a hand over actionable Explorer and
Settings items, and returns to the selected pointer theme outside those areas.
TextEdit and Settings also offer a vertical-bar or underline insertion caret,
stored in `/etc/cursors/type` and applied in the editor.
Users can create a personal layout with `keymap create NAME.lyt`, edit the
resulting file in `~/.config/keyboards/` with TextEdit, then select it with
`keymap use NAME.lyt`. Personal layouts apply to that user's active console;
the built-in `keymap us.lyt` and `keymap fr-azerty.lyt` selections remain
system-wide.
The built-in applications are separate projects with per-app Makefiles and
share the console UI Kit. See [application development](./app-development.md)
for building `.APPL` files and dropping custom applications into `/app/custom/`.
The Settings application is available as `/app/system/bin/settings.APPL` and
through its `/app/system/settings.link` shortcut. It exposes keyboard, cursor,
and DHCP-autostart controls, whose effects are implemented. On systems whose
ACPI firmware supplies a usable S3 state and waking vector, it also exposes
automatic suspend choices of disabled, 5, 10, 15, or 30 minutes. The default
`/etc/power.conf` value is 15 minutes; changes apply immediately and are
persistent. The idle timeout is polled while either graphical or serial
interactive input is waiting. The QEMU enter/wake/resume and automatic-idle paths were previously validated.
Their test scripts were removed with the `tests/` directory, so those checks
cannot currently be rerun from this checkout. Automatic suspension is
not a background timer service: it is checked while CreateOS is waiting for
interactive input. It will not trigger while a program is continuously
computing or otherwise not waiting for input. Keyboard and power-button wake
and real-hardware behavior still require validation. Language and time-zone preferences are intentionally
omitted until translations and time-zone conversion are implemented.
Keyboard selection is saved to the signed-in user's
`~/.config/keyboards/default` (and takes precedence over the system default in
`/etc/keyboards/default`) and applied immediately to that account's active
console. Cursor selection is saved to `/etc/cursors/default` and applied
immediately. DHCP autostart is stored in `/etc/network.conf` and takes effect
on the next system startup. Unreadable or invalid settings are shown as
unavailable rather than silently guessed or omitted. Save/apply failures are
reported in the application. Cursor sources are bounded to 16x16 monochrome
sprites, displayed at their native size, and kept inside the framebuffer
edges. Full console redraws preserve the cursor overlay. The optional Noto Sans
TrueType font now uses 16x20-pixel terminal cells and 16-pixel glyph rendering;
the bitmap font remains available as a fallback. The built-in 8x8 font includes
common CP437 French accented
letters (`Ç`, `ü`, `é`, `â`, `ä`, `à`, `å`, `ç`, `ê`, `ë`, `è`, `ï`, `î`,
`ì`, `Ä`, `Å`); the French AZERTY layout emits the commonly used lowercase
accented forms. Mouse deltas are
accumulated between text cells to avoid losing small movements.
Building either ISO runs `refresh-system-apps`, which updates Settings,
TextEdit, Explorer, the `lspci` hardware-diagnostics command, and the French
keyboard layout in the existing `build/disk.img` before embedding it. The `clock`
command reports monotonic elapsed time using HPET when available and the PIT
tick clock otherwise. This preserves accounts and other filesystem data; use
`make disk-image` only when an intentional full filesystem rebuild is wanted.

When Explorer opens a native ELF executable, it asks before running it unless
it is an installed `.APPL` directly in `/app/system/bin` or a valid personal
application shortcut. A `.APPL` suffix alone does not bypass the prompt. The
confirmation is an accidental-launch warning only: it does not verify, restrict,
or sandbox the program. Installed `.APPL` applications and shell commands use
the same console-scoped user identity and kernel syscall checks. Filesystem
ownership checks separate home trees between accounts, but they do not separate
applications running as the same account. CreateOS does not yet provide
per-process containers. The x86-64 scheduler currently switches only
cooperative kernel tasks; ring-3 applications still share console-scoped
credentials, file descriptors, and syscall state, and timer preemption is not
enabled. User image, stack, and heap mappings occupy a 1–2 GiB virtual window
that is omitted from each application's kernel identity map, preventing those
mappings from aliasing kernel low-memory mappings. This is a memory-layout
safety boundary, not container-style isolation. Ownership migration and
cross-account filesystem denial still need dedicated QEMU tests before the
filesystem boundary is considered fully validated.
Each successful user-image transition advances a per-console execution
generation. POSIX descriptors and TCP stream handles are bound to that
generation, so a replacement application cannot reuse the previous
application's handles; the previous process's TCP stream is closed during the
transition. QEMU tests cover descriptor revocation and stale-generation
rejection; closing a live TCP stream across a transition still needs a
networking integration test. This does not give applications separate
credentials, working directories, address-space lifetimes, or schedulable
process contexts.

Outbound ICMP, DNS, and TCP client syscalls are currently restricted to an
authenticated administrator account; ordinary accounts cannot use these
network clients. Device inspection and DHCP configuration remain available to
system startup. This is a coarse account-role gate, not a per-application
network capability system or an inbound packet firewall. Reboot and shutdown
requests through the process-exit syscall also require an administrator.
Administrative syscalls additionally require the active executable to have
been launched from `/bin`, `/sbin`, `/usr/bin`, or `/app/system/bin`. Executables
from a user's home directory therefore cannot inherit the session's
administrator syscalls. This is a path-based restriction, not a signed binary
or per-process capability system; system-directory applications remain
trusted. User-requested `exec` also cannot acquire system privileges solely
from its target path: it retains system privileges only when the current
application already has them and the target remains in a protected system
directory. Exit and user-fault recovery preserve the current application's
privilege state instead of granting it based on the recovery program's path.
Because the kernel does not yet save parent ring-3 contexts, returning from a
user-owned application may leave the replacement shell without administrative
privileges until the session is restarted. The single active ring-3 context
still has console-scoped state.

While credentials are still console-scoped, the login syscall refuses to
replace an already-authenticated identity. `passwd` verifies the current
account's password without changing that identity, and the password-change
syscall also requires the current password for non-administrators. `sudo` cannot
safely switch to another account in this architecture; it only runs a
same-identity command for an already-administrative account and otherwise
reports that limitation.

First boot also records a manual application-language and region selection in
`/etc/locale.conf`, and a time-zone name in `/etc/timezone`. No network
geolocation or external API is used. These are configuration preferences only:
translated application catalogs and time-zone conversion are not implemented
yet, so the selected language and time zone do not currently change displayed
text or the value returned by the clock syscall.

## Scripts, C compiler, and SVG

The files in the repository-root [`scripts/`](../scripts/) directory are host
build, dependency, and source-fetch helpers; they are not guest shell scripts
and should not be executed by the guest. The current shell does not implement
a script interpreter. The repository-root `Makefile`, NASM, cross GCC/binutils,
and `grub-mkrescue` run on the build host. GRUB itself boots the ISO, but
`grub-mkrescue` is not a guest program. Guest NASM, GNU make, TCC compilation,
a C compiler command, and SVG viewing are not currently complete or installed
features. Guest TinyCC is built separately from the native host compiler by
the rootfs build; `make toolchain-tcc` still builds only the host utility.

The documentation does not describe unimplemented features as working.
