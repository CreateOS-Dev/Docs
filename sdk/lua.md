# Lua on CreateOS

This guide teaches Lua as it is used in the CreateOS guest. It begins with
the language, then explains how the bundled interpreter, bytecode compiler,
filesystem, and process limitations affect real scripts.

CreateOS includes Lua 5.4.8 as a native userland application. The upstream
source lives at `kernel/lib/third_party/lua-5.4.8/`; CreateOS compatibility
code and build rules live in `rootfs_maker/root/app/lua/`. You do not need to
download a separate interpreter to write scripts for a built image.

Lua scripts are interpreted by `lua`. `luac` compiles them to Lua bytecode.
Bytecode is still input to the Lua interpreter: it is not an ELF program,
does not have the `.APPL` executable format, and cannot be launched as a
native application by itself.

## Contents

- [Run the interpreter](#run-the-interpreter)
- [Your first script](#your-first-script)
- [Values and variables](#values-and-variables)
- [Expressions and operators](#expressions-and-operators)
- [Branches and loops](#branches-and-loops)
- [Functions](#functions)
- [Tables](#tables)
- [Strings and text](#strings-and-text)
- [Files](#files)
- [Modules](#modules)
- [Errors and debugging](#errors-and-debugging)
- [Interactive mode](#interactive-mode)
- [Lua bytecode](#lua-bytecode)
- [CreateOS runtime boundaries](#createos-runtime-boundaries)
- [A small complete project](#a-small-complete-project)
- [Exercises](#exercises)

## Run the interpreter

In the guest terminal, verify that both commands are installed:

```text
lua -v
luac -v
```

Run an expression without creating a file:

```text
lua -e print("hello")
```

The guest shell splits command arguments on whitespace and does not
understand shell quotes. Keep `-e` code to one token (no spaces), or enter
the expression at the Lua prompt.

Run a script by path:

```text
lua /home/root/hello.lua
```

Starting `lua` without a script enters its interactive prompt. The prompt
accepts multiline chunks. Enter `exit` or press Ctrl-C to leave. The line
buffer for a single REPL input line is 8 KiB.

The first boot creates a home directory for the initial account. Scripts
stored there remain on the CFS5 disk across boots, unless the image is
recreated. Back up `build/disk.img` before rebuilding it if those files
matter.

## Your first script

Create `/home/root/hello.lua` in TextEdit or another guest editor:

```lua
local name = "CreateOS"
print("Hello, " .. name)

for step = 1, 3 do
    print("step " .. step)
end
```

Then run it:

```text
lua /home/root/hello.lua
```

The keyword `local` creates a local variable. The `..` operator concatenates
strings. The `for` loop repeats with values 1, 2, and 3. `print` is provided
by Lua's base library.

Keep one small task per script while learning. That makes it easier to tell
whether an error comes from Lua syntax, the guest runtime, a bad path, or
an unavailable operating-system operation.

## Values and variables

Lua is dynamically typed: values have types, while variables can refer to
values of different types at different times.

```lua
local count = 4
local title = "Files"
local enabled = true
local nothing = nil
```

The main value types are:

| Type | Example | Notes |
| --- | --- | --- |
| `nil` | `local value = nil` | Absence of a value; assigning `nil` removes a table field. |
| boolean | `true`, `false` | Only `false` and `nil` are false in conditions. |
| number | `12`, `3.5` | Lua 5.4 has integer and floating-point numeric values. |
| string | `"hello"` | Immutable byte strings; the standard library includes UTF-8 helpers. |
| table | `{ name = "Ada" }` | Associative collection used for arrays, maps, and records. |
| function | `function() end` | Functions are values and may be passed to other functions. |
| thread | `coroutine.create(f)` | A Lua coroutine; it is not an OS process. |
| userdata | runtime-provided values | No CreateOS native-module ABI is exposed to load arbitrary C userdata. |

Use `type(value)` when a function accepts more than one kind of value:

```lua
local function describe(value)
    return type(value)
end
```

### Local and global names

Declare variables with `local` unless a global is intentionally part of a
module interface:

```lua
local current_path = "/home/root"

local function is_home(path)
    return path == current_path
end
```

An undeclared assignment creates or changes a global. In a larger script,
that can accidentally overwrite a value used elsewhere. Keep helper
variables local and return public values from modules.

### Multiple assignment

Lua assigns several values at once:

```lua
local first, second = "left", "right"
first, second = second, first
```

Functions can return several results. Capture only the results you need:

```lua
local ok, message = pcall(function()
    return "finished"
end)
```

## Expressions and operators

Arithmetic operators are `+`, `-`, `*`, `/`, floor division `//`, modulo
`%`, and exponentiation `^`. The unary minus negates a number.

```lua
local pages = 17 // 5
local remainder = 17 % 5
local square = 4 ^ 2
```

String concatenation uses `..`, not `+`. Compare values with `==`, `~=`,
`<`, `<=`, `>`, and `>=`.

Logical operators are `and`, `or`, and `not`. They short-circuit and return
one of their operands; they do not always return the boolean `true` or
`false`.

```lua
local label = supplied_label or "Untitled"
local allowed = user and user.enabled
```

Only `false` and `nil` are false. In particular, zero and the empty string
are both true:

```lua
if 0 then
    print("This branch runs")
end
```

For a boolean test, write the intended condition explicitly. Do not port
truthiness assumptions from C, Python, or a shell language.

Lua's length operator `#` is straightforward for a sequence with no holes:

```lua
local colors = { "blue", "green", "white" }
print(#colors)
```

Do not use `#table` to count arbitrary map keys. A map may not have a
sequence boundary that represents its number of entries; iterate with
`pairs` when you need to visit a map.

## Branches and loops

### Conditional branches

```lua
if count == 0 then
    print("empty")
elseif count < 0 then
    print("invalid")
else
    print("items: " .. count)
end
```

The condition ends with `then`; the complete branch ends with `end`.

### While loop

```lua
local remaining = 3
while remaining > 0 do
    print(remaining)
    remaining = remaining - 1
end
```

Check that each loop can make progress toward its end. A loop whose
condition never changes will keep the interpreter busy.

### Repeat loop

```lua
local attempts = 0
repeat
    attempts = attempts + 1
    print("attempt", attempts)
until attempts == 3
```

The body runs at least once. Here the loop stops after its third iteration.

### Numeric for loop

```lua
for index = 1, 5 do
    print(index)
end

for value = 10, 2, -2 do
    print(value)
end
```

The optional third expression is the step. A positive step counts upward;
a negative step counts downward.

### Generic for loop

Use `ipairs` for the contiguous sequence part of a table and `pairs` for
key/value entries:

```lua
local commands = { "ls", "pwd", "help" }
for index, command in ipairs(commands) do
    print(index .. ": " .. command)
end

local settings = { theme = "dark", rows = 25 }
for key, value in pairs(settings) do
    print(key .. " = " .. tostring(value))
end
```

`pairs` does not promise a useful display order. Sort keys separately if
the order is important to the user.

## Functions

Define and call a function:

```lua
local function clamp(value, low, high)
    if value < low then
        return low
    elseif value > high then
        return high
    end
    return value
end

print(clamp(12, 0, 10))
```

Functions can return more than one value:

```lua
local function split_once(text, separator)
    local start, finish = string.find(text, separator, 1, true)
    if not start then
        return text, nil
    end
    return string.sub(text, 1, start - 1),
           string.sub(text, finish + 1)
end
```

Functions are values. A function can be stored in a table, passed as an
argument, or returned from another function. Closures can retain local
variables from the scope where they were created:

```lua
local function make_counter()
    local value = 0
    return function()
        value = value + 1
        return value
    end
end

local next_number = make_counter()
print(next_number())
print(next_number())
```

Use `...` for a variable number of arguments only when the call really
needs it. The extra flexibility can make APIs harder to understand.

## Tables

Tables are Lua's main compound data type. They can represent ordered
sequences, maps, and records.

### Sequences

```lua
local files = { "readme.txt", "notes.lua" }
files[#files + 1] = "todo.txt"

for index, filename in ipairs(files) do
    print(index, filename)
end
```

Sequence positions conventionally start at 1. This differs from C arrays,
which normally start at index 0.

### Maps and records

```lua
local app = {
    name = "viewer",
    version = "1.0",
    enabled = true
}

print(app.name)
app.enabled = false
app["last run"] = "today"
```

Bracket syntax accepts expressions and keys that are not identifier names:

```lua
local values = {}
local key = "current"
values[key] = 42
```

### Nested values

```lua
local desktop = {
    width = 80,
    height = 25,
    cursor = { x = 0, y = 0 }
}

desktop.cursor.x = desktop.cursor.x + 1
```

Tables are references. Assigning one table to another variable does not
copy its contents:

```lua
local original = { count = 1 }
local alias = original
alias.count = 2
print(original.count) -- 2
```

Make an explicit copy if independent mutation is required.

## Strings and text

Strings are immutable. Common operations live in the `string` library:

```lua
local message = "CreateOS"
print(string.len(message))
print(string.sub(message, 1, 6))
print(string.upper(message))
```

The length operator `#message` also returns the byte length. In UTF-8 text,
byte length and visible-character count can differ. The UTF-8 library
provides helpers for code points, but terminal rendering and cursor movement
are still governed by CreateOS's text console.

Lua patterns are not POSIX regular expressions. For literal matching, pass
`true` as the plain-search argument to `string.find`:

```lua
local start = string.find("a+b", "+", 1, true)
```

This avoids interpreting `+` as a pattern operator.

Use `string.format` for formatting:

```lua
local line = string.format("row=%d, label=%s", 3, "menu")
print(line)
```

Formatting is supplied by the Lua runtime and is separate from the limited
C `printf` formatter.

## Files

The guest opens scripts from the CFS5 filesystem. A Lua script may use the
standard `io` library, but all paths still refer to guest paths, and
filesystem permissions still apply.

The following is a cautious text-file pattern:

```lua
local path = "/home/root/message.txt"
local file, open_error = io.open(path, "r")
if not file then
    io.stderr:write("cannot open file: ", tostring(open_error), "\n")
    return
end

local contents = file:read("*a")
file:close()
print(contents)
```

For a bounded read, prefer an explicit byte limit:

```lua
local file = assert(io.open("/home/root/message.txt", "r"))
local first_line = file:read("*l")
file:close()
print(first_line or "(empty file)")
```

Close files even when later processing fails. Check the return value of
`io.open`; do not assume the path exists or that the user has permission.
File APIs are provided by the Lua compatibility layer and userland, so a
desktop Lua program that depends on host-specific file descriptors or
special devices may need changes.

### Paths and script size

The app loads script files no larger than 1 MiB. Module lookup checks the
current directory, the active user's home, and `/usr/share/lua/5.4`. It
looks for both `name.lua` and `name/init.lua` patterns at those roots.

Keep project modules under one of those locations or add a guest path to
`package.path`:

```lua
package.path = package.path .. ";/home/root/myapp/?.lua"
```

The `package.cpath` used for native C modules is deliberately empty. The
runtime does not load arbitrary native extension modules.

## Modules

A module is a Lua file that returns a value. Example
`/home/root/greetings.lua`:

```lua
local M = {}

function M.hello(name)
    return "Hello, " .. name
end

return M
```

Use it from another script:

```lua
local greetings = require("greetings")
print(greetings.hello("CreateOS"))
```

Module names map to paths by replacing dots with directory separators.
`require("tools.text")` searches for a path such as
`tools/text.lua` beneath each configured module root.

Keep module initialization small. `require` caches the returned module
value, so subsequent requires in the same Lua state normally reuse it.

## Errors and debugging

Lua errors include syntax errors detected while loading and runtime errors
raised during execution. The interpreter reports a message and returns a
nonzero result for a failed script.

Use `assert` to stop when a required condition is false:

```lua
local number = tonumber("42")
assert(number, "expected an integer")
```

Use `pcall` when the script can recover from a particular operation:

```lua
local ok, result = pcall(function()
    return do_work()
end)

if not ok then
    io.stderr:write("operation failed: ", tostring(result), "\n")
end
```

Do not wrap the entire program in a catch-all that hides a failure. Report
enough context to find the bad input or operation.

For debugging, print the value and its type:

```lua
print("value:", tostring(value), "type:", type(value))
```

Use short scripts to isolate an issue. First verify that the interpreter
starts, then test file access, module loading, and the feature that failed.
This separates language errors from filesystem and compatibility-layer
errors.

The debug library is opened, but its presence does not guarantee that a
desktop debugger, terminal integration, or every debug hook works exactly
as it does on a host OS.

## Interactive mode

The REPL reads an input line, compiles it as a Lua chunk, and runs it. A
syntactically incomplete chunk can continue on the next line. It is useful
for trying expressions:

```text
> local total = 8 + 5
> print(total)
13
```

The REPL has a fixed line-buffer capacity of 8 KiB and uses the UI Kit's
terminal input path. It is not an editor and does not provide shell
quoting, pipes, or command substitution. Use TextEdit to save a longer
program to a `.lua` file.

## Lua bytecode

Compile a script:

```text
luac -o /home/root/hello.luac /home/root/hello.lua
```

Run the compiled chunk through Lua:

```text
lua /home/root/hello.luac
```

Bytecode is tied to Lua's bytecode format and interpreter version. Keep
source alongside it when you need to edit or rebuild a script. Bytecode is
not encrypted and should not be treated as a security boundary.

## CreateOS runtime boundaries

The runtime opens the base, coroutine, debug, I/O, math, OS, package,
string, table, and UTF-8 libraries. This is the set of Lua library modules
the app initializes; it does not imply that every operation has a desktop
operating-system service behind it.

| Facility | CreateOS behavior |
| --- | --- |
| `os.execute` | Does not start another program. |
| `io.popen` | Does not create a process pipe. |
| `os.getenv` | Exposes only `HOME`, `PWD`, and `PATH`. |
| Native `package.cpath` modules | Disabled; the configured path is empty. |
| Calendar functions | Use UTC. |
| Script file size | Maximum 1 MiB for the app's script loader. |
| REPL input line | Fixed 8 KiB buffer. |
| `luac` output | Lua bytecode, not native `.APPL` machine code. |

If a script depends on another operating-system facility, test it in the
guest rather than assuming the host version's behavior. Use C and the SDK
when the feature requires a CreateOS syscall or native app integration.

## A small complete project

This example reads a known guest path, handles an open failure, and closes
the file before returning. The Lua application currently loads a script
path but does not populate the usual command-line `arg` table:

```lua
local path = "/home/root/notes.txt"
local file, message = io.open(path, "r")
if not file then
    io.stderr:write("preview: ", tostring(message), "\n")
    return
end

local line_number = 0
while true do
    local line = file:read("*l")
    if not line then
        break
    end
    line_number = line_number + 1
    if line_number > 20 then
        break
    end
    print(line_number .. "  " .. line)
end

file:close()
```

Save the script as `/home/root/preview.lua` and run
`lua /home/root/preview.lua`. Replace `path` with a file that exists on the
guest disk. The CFS5 paths are not host paths.

## Exercises

1. Change the greeting to read the name from a local variable.
2. Make a table of three favorite commands and print each with `ipairs`.
3. Write a function that returns the first and last character of a string.
4. Write a module that returns a table with two helper functions.
5. Count lines in a text file while handling a failed open.
6. Compile the script with `luac`, then run the bytecode with `lua`.
7. Deliberately misspell a variable, read the error, and fix it.
8. Try to call a disabled process API and record the actual runtime error.

For the app's build rules and exact source locations, see
`rootfs_maker/root/app/lua/README.md`. For app development, see
[the SDK guide](app-development.md); for language status, see
[the language index](languages.md).
