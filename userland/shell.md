# Using the CreateOS shell

The CreateOS shell is the command prompt used to navigate the guest and start
programs. It is a small native command runner, not Bash, POSIX `sh`, or a
general shell-script interpreter. Treat each entered line as one program
name followed by whitespace-separated arguments.

This guide documents the current shell behavior: command lookup, arguments,
line editing, history, completion, and the boundary between the shell and
the separately compiled command programs.

## Contents

- [A first session](#a-first-session)
- [Command line format](#command-line-format)
- [How commands are found](#how-commands-are-found)
- [Applications and commands](#applications-and-commands)
- [Line editing](#line-editing)
- [History and completion](#history-and-completion)
- [Paths and working directory](#paths-and-working-directory)
- [Available commands](#available-commands)
- [What this shell does not implement](#what-this-shell-does-not-implement)
- [Writing a guest command](#writing-a-guest-command)
- [Troubleshooting](#troubleshooting)

## A first session

After login, type one command and press Enter:

```text
help
pwd
ls /
cd /home/root
ls
```

The exact programs in an image depend on which applications were staged.
`help` lists installed command files, and individual commands generally
print a usage message when called with unsupported arguments.

`cd`, `exit`, `help`, and `tty` are programs in `/bin`, not parser keywords
or a built-in shell scripting language. In particular, the shell command
parser itself does not implement control flow or execute a sequence of
commands from a file.

## Command line format

The input line is split at spaces and tabs. The first token is the program
name; the rest become its arguments. For example:

```text
echo hello CreateOS
```

is passed as a program named `echo` with two arguments, `hello` and
`CreateOS`.

The shell line buffer is 256 bytes including its terminating zero, so a
line can contain at most 255 input bytes. The command parser stores at most
31 arguments plus the terminating null pointer. The kernel validates
executable paths and argument vectors as well.

### Spaces inside arguments

The parser does not recognize quoting or escaping:

```text
echo "two words"
```

does not make one argument named `two words`; the quote characters are
ordinary characters and the space still separates arguments. There is no
portable way in the current shell to enter an argument containing a space.
Use a path without spaces or write a program that asks for the value after
it has started.

### Symbols are ordinary characters

The shell does not interpret `;`, `&`, `|`, `<`, `>`, `$`, `*`, backticks,
or parentheses as shell operators or expansions. They remain characters
inside a token. For example:

```text
echo first; echo second
```

does not run two commands; it attempts one command line whose tokens include
`first;`.

There is no command substitution, variable expansion, globbing, pipeline,
redirection, background job, or shell script execution. Do not paste a Bash
recipe into the guest and expect its control operators to work.

## How commands are found

A command name without a slash is searched under `/bin` and `/usr/bin`.
When the name has no `.bin` suffix, the shell first tries the `.bin` form
and then the unsuffixed form. A name containing a slash is treated as a
path; if it has no `.bin` suffix, the shell also tries the path with
`.bin` appended.

Examples of command names:

```text
ls
ls.bin
/bin/ls.bin
```

Only executable files are accepted. If no suitable file can be opened, the
shell prints `command not found`. Applications with `.APPL` or `.appl`
suffixes are not run as ordinary commands; the shell directs the user to
the `open` program.

The lookup rules are implemented in
`rootfs_maker/root/prog/shell/shell_exec.c`. Use that source when changing
the resolution order or executable suffix rules.

## Applications and commands

Guest command programs are normally staged under `/bin` or `/usr/bin` and
end with `.bin`. Native apps are staged as `.APPL` files. Launch an app
through the installed application command:

```text
open Explorer.APPL
open TextEdit.APPL /home/root/notes.txt
```

Explorer can also browse to an application. A custom `.APPL` placed in
`/app/custom/` can be launched there.

The shell is itself a guest program. Commands are separate guest programs,
not shell functions. This makes the command implementation and its
permissions visible in the source tree, but it also means there is no
interactive shell language for composing commands.

## Line editing

The prompt edits the current input line before it is submitted:

| Key/action | Current behavior |
| --- | --- |
| Left / Right | Move the cursor within the line. |
| Backspace | Remove the character before the cursor. |
| Delete | Remove the character under the cursor. |
| Shift+Left / Shift+Right | Extend a character selection. |
| Up / Down | Browse command history, except while cycling completion choices. |
| Tab | Complete a command or path; repeated Tab can show/select matches. |
| Shift+Tab | Cycle completion choices in the reverse direction. |
| Mouse click and drag | Position the cursor and select input text. |
| Ctrl+C | Cancel an unselected input line; copy selected input when text is selected. |
| Ctrl+X | Copy and delete selected input. |
| Ctrl+V | Paste clipboard text if it fits in the input buffer. |
| Enter | Submit the current line. |

Some special keys depend on the active terminal and keyboard layout. If a
key appears as an ordinary character instead of an editing action, check the
selected layout in Settings.

The shell also displays a blinking input caret. Its shape can be selected
through `/etc/cursors/type`; the current values `underline`, `underscore`,
`_`, or `line` select an underscore, while `bar`, `|`, or `vertical` select
a vertical bar.

## History and completion

The shell remembers commands and supports Up/Down navigation. The `history`
program explains that history navigation is performed with those keys.
Repeated entries may be suppressed by the history implementation.

Tab completion considers names from `/bin` and `/usr/bin` when completing
commands, and entries in directories when completing paths. When several
names share a prefix, the shell may fill the common prefix and display a
completion menu; use Tab, Shift+Tab, or Up/Down to cycle the available
choices, then continue typing or press Enter.

Completion is a convenience for the current line, not a filesystem search
utility. If the expected file is not suggested, type its full path and use
`ls` to check that it exists.

## Paths and working directory

`pwd` prints the current working directory. `cd` changes it:

```text
pwd
cd /home/root
pwd
```

Absolute paths begin with `/`. Relative paths are interpreted from the
current working directory. The root filesystem uses CreateOS's CFS5
filesystem, and normal account permissions still apply.

There is no host-side current-directory sharing. A path such as
`/home/root/file.txt` is a path inside the guest, not a path on macOS or
Linux. The staged files under `build/rootfs/` are build inputs; the mounted
guest disk is `build/disk.img`.

## Available commands

Use `help` inside the guest for the exact command set present in that image.
The userland source has separately built programs for common tasks such as:

| Task | Example command |
| --- | --- |
| List files | `ls /home/root` |
| Print a file | `cat /home/root/readme.txt` |
| Show working directory | `pwd` |
| Change working directory | `cd /home/root` |
| Make a directory | `mkdir /home/root/projects` |
| Create an empty file | `touch /home/root/notes.txt` |
| Copy or move files | `cp SOURCE DEST`; `mv SOURCE DEST` |
| Remove a file | `rm /home/root/notes.txt` |
| Show a file prefix or tail | `head PATH`; `tail PATH` |
| Print text | `echo hello` |
| Show command history help | `history` |
| List installed commands | `help` |
| Open an application | `open Explorer.APPL` |
| Run Lua | `lua /home/root/script.lua` |
| Compile Lua bytecode | `luac -o /home/root/script.luac /home/root/script.lua` |
| Compile C | `tcc -o /home/root/program /home/root/program.c` |
| Show network interfaces | `net` or `ifconfig` |
| Show kernel version | `uname` |
| Switch terminal | `tty tty1` |

Command arguments are subject to the whitespace-only parsing rules above.
If a command accepts an option or path, check its usage text and the
matching source in `rootfs_maker/root/commands/`.

## What this shell does not implement

The current command prompt does not implement:

- shell scripts (`.sh`);
- variables or environment assignment syntax;
- quoted strings or backslash escaping;
- `;`, `&&`, `||`, pipelines, or redirection;
- wildcard expansion;
- command substitution;
- background jobs or job control;
- Bash functions, loops, conditionals, or aliases.

`lua` can run a Lua script, and TinyCC can compile a C source file. Neither
turns the shell into Bash. For repeatable multi-step host builds, use the
Makefiles in the source tree. For guest automation, write a Lua script for
operations supported by the runtime or create a native application.

## Writing a guest command

The command source files live in `rootfs_maker/root/commands/`. A command is
a C program with a `main` function, built by the rootfs-maker Makefile and
staged as a `.bin`.

A small command should:

1. Validate `argc` and every required argument before using them.
2. Print a concise usage message for invalid invocation.
3. Check every syscall or library return value.
4. Return a meaningful status code.
5. Avoid assumptions about desktop POSIX behavior.

For example, commands can use the CreateOS user syscall interface in
`rootfs_maker/lib/user_syscall.h`. Use the cross-toolchain before building:

```sh
make toolchain
make -C rootfs_maker
make userapps
```

When adding a command, check how `CMD_SRCS` and `CMD_NAMES` are constructed
in `rootfs_maker/Makefile`, then rebuild the staged image using the
top-level workflow. The program should not parse a shell language unless
that parser is intentionally part of the program itself.

## Troubleshooting

### `command not found`

Check the spelling, use `help`, and confirm the command was included in the
image. Commands are looked up in `/bin` and `/usr/bin`; a file merely
present in a host checkout is not necessarily installed in the current
guest disk.

### `open` tells me to use the app launcher

Use the `open` command for `.APPL` files. A native application is not a
shell command binary.

### Quotes do not group words

That is expected: the line parser splits on spaces and tabs without a
quoting state. Remove the spaces, use a different input path, or pass data
through a file.

### A command runs but my following command does not

There is no command separator. Enter each command on its own line.

### A shell script prints syntax errors

The shell does not execute `.sh` files. Run a supported interpreter on its
own script type, for example `lua /home/root/example.lua`, or implement the
operation as a native command.

For account, app, and filesystem behavior, continue with the
[userland guide](userland.md). For application source, see
[C programming](../sdk/c.md) and [SDK app development](../sdk/app-development.md).
