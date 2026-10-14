# Languages in CreateOS

The guest can run native x86-64 programs and Lua scripts. Other languages
need a runtime and a port that matches the kernel's executable and system-call
interfaces; a compiler installed on the host does not provide that runtime.

| Language | Available now | Build or run |
| --- | --- | --- |
| C | Host SDK cross-build and guest TinyCC | Build `.APPL` with `sdk/app.mk`, or use `tcc` in the guest. |
| Lua 5.4.8 | Guest interpreter and bytecode compiler | `lua script.lua`; compile with `luac`. |
| x86-64 assembly | Assembly in kernel and SDK apps | SDK apps can include uppercase `.S` sources. |
| CreateOS shell | Small interactive command interpreter | Enter commands at the guest prompt; it is not Bash. |
| Python | Host-side helper scripts only | No guest Python interpreter is provided. |
| C++, Rust, Go, Java, JavaScript | No maintained guest runtime/toolchain | Host-built programs are not runnable in CreateOS by default. |

The detailed guides for the supported programming languages are
[C](c.md), [Lua](lua.md), and [x86-64 assembly](assembly.md). The guest's
interactive command interface is documented separately in
[Shell commands](../userland/shell.md); it is not a shell scripting runtime.

## C

For an SDK project or the included examples, build the pinned cross-toolchain
and use the SDK:

```sh
make toolchain
./sdk/scripts/build-example.sh hello
./sdk/scripts/build-example.sh uikit-demo
```

The SDK makes static `.APPL` executables for CreateOS. Guest TinyCC is useful
for short programs, but currently has no dynamic linker or `-run` mode:

```text
tcc -o /home/root/hello hello.c
chmod u+x /home/root/hello
/home/root/hello
```

See [application development](app-development.md) for project setup, libraries,
and installing an app in the image.

## Lua

Lua 5.4.8 and `luac` are already built into the guest userland. The upstream
source used by the component build lives at
`kernel/lib/third_party/lua-5.4.8/`; CreateOS compatibility code is in
`rootfs_maker/root/app/lua/`. There is no separate Lua installation needed
for guest scripts.

```text
lua -v
lua /home/root/hello.lua
luac -o /home/root/hello.luac /home/root/hello.lua
lua /home/root/hello.luac
```

Read the [Lua guide](lua.md) for libraries, module paths, limits, and behavior
that differs from desktop Lua.

## x86-64 assembly

The SDK accepts uppercase `.S` files recursively under an app's `src/`
directory. It compiles them with the x86-64 cross-GCC and links them into the
same `.APPL` as C code. The SDK expects GNU assembler (GAS) syntax for these
app files. The app linker supplies the CreateOS startup entry; assembly
functions should normally be called from the C `main`, rather than defining
another `_start`.

See the [assembly guide](assembly.md) for a working C/GAS example and the
difference between SDK assembly and the NASM sources used in boot code.

## Languages not bundled

There is no guest Python, C++, Rust, Go, Java, or JavaScript runtime in this
tree. Do not copy a host executable into the root filesystem and expect it to
run: it must target CreateOS's executable format and ABI, and any interpreted
language also needs its interpreter and supporting libraries. Porting another
language means maintaining those pieces and testing them against this kernel.
