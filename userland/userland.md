# Using the userland

After startup, CreateOS shows a login prompt. The first boot runs a short setup
for language, keyboard, region, time zone, and the administrator account.
After that, sign in with the account created during setup.

## The shell

The shell is a small command runner built into the userland. Commands are
loaded from `/bin` and `/usr/bin`; guest command files commonly end in
`.bin`. Applications use `.APPL` and are opened with `open`.

Examples:

```text
help
ls /
cd /home/root
pwd
open Explorer.APPL
open TextEdit.APPL /home/root/notes.txt
```

The parser is intentionally simple. It splits input on whitespace and does
not implement Bash quoting, pipes, redirects, variable expansion, or job
control. The shell has line editing, history, and completion for commands
and paths.

Use `help` in the guest for commands available in the current image. Some
commands require administrator access.

## Accounts and home directories

First boot creates the initial administrator account. Additional accounts
can be managed with the account commands shown by `help`. Account names are
currently restricted to ASCII letters. Each account has a home directory and
an `Applications` folder containing shortcuts to the system apps.

File permissions are checked by the kernel. Normal users can work in their
own home directory; administrative operations require the appropriate
privileges. The user database and home files live on the CFS5 disk, so keep a
backup before changing them by hand.

## Applications

Explorer opens directories, starts installed apps, and can install `.APPL`
files into the current user's personal application directory. The
system-provided applications include:

- **Explorer** for browsing files and launching apps.
- **TextEdit** for editing text and source files.
- **Settings** for keyboard, cursor, networking, and power preferences.
- **Lua** and **luac** for running Lua scripts and compiling Lua bytecode.
- **TinyCC** for building static native C programs in the guest.

TextEdit is a text-mode application, not a graphical desktop editor. It
supports multiple tabs and basic source highlighting. See the app's source at
`rootfs_maker/root/app/textedit/`.

An `.APPL` is a native x86-64 ELF program, not a sandboxed package. Only run
programs you trust.

## C and TinyCC

The guest TinyCC build is installed as `/app/system/bin/tcc.APPL`; the
`tcc` command forwards to it. It uses headers in `/usr/include` and static
runtime files in `/usr/lib/tcc`.

For example:

```text
tcc -o /home/root/hello hello.c
chmod u+x /home/root/hello
/home/root/hello
```

The guest compiler targets CreateOS and produces static executables. Dynamic
linking and TinyCC's `-run` mode are not available. The host TinyCC build is a
separate development tool and is not the guest compiler.

## Lua

Lua 5.4.8 is available as `lua`; the bytecode compiler is `luac`.

```text
lua -v
lua -e print("hello")
lua /home/root/script.lua
luac -o /home/root/script.luac /home/root/script.lua
lua /home/root/script.luac
```

The runtime includes the standard Lua libraries, but it is not a full POSIX
environment. Native C modules are disabled; `os.execute` and `io.popen` do
not launch processes. File loading and `package.path` are described in the
Lua project notes at `rootfs_maker/root/app/lua/README.md`.
The guest shell does not group quoted arguments, so `-e` code must be one
whitespace-free argument; use the interactive prompt for longer expressions.

## Networking

DHCP autostart is controlled by `/etc/network.conf`. The default image
enables it. The `net`, `ping`, and `curl` commands provide the network tools
available in the current build. `curl URL` displays a response in the
terminal; `curl -o PATH URL` downloads a small response into the current
user's home after confirmation.

HTTPS needs a working clock, hardware entropy, and a trusted certificate
bundle. HTTPS errors are not a reason to disable certificate checks.

The guest shell does not run `.sh` files. Startup networking is handled by
compiled userland code, not a shell script.

## Files and applications

Built-in commands are staged under `/bin`; system apps live in
`/app/system/bin`; custom apps are staged under `/app/custom`. A `.link` file
under `/app/system/` is a shortcut used by the app launcher.

To add a native app from the host, copy its `.APPL` file into
`rootfs_maker/root/appcustom/`, then run `make userapps` and rebuild the disk
image. The [SDK guide](../sdk/app-development.md) covers building apps.

The root filesystem is CFS5. It is not mounted as a regular host directory.
Use the CFS5 host tools or the guest filesystem commands to inspect it.
