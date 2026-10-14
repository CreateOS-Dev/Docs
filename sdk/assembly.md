# x86-64 assembly in apps and the kernel

CreateOS uses assembly at more than one layer. SDK app sources and low-level
boot/kernel sources do not all use the same assembler syntax.

## Assembly in an SDK app

Place uppercase `.S` files under the app's `src/` directory. The SDK finds
them recursively, compiles them with the x86-64 cross-GCC, and links the
objects into the app alongside its C sources. These files use GNU assembler
(GAS) syntax and are preprocessed by GCC.

Example `src/math.S`:

```asm
.text
.globl add_one
.type add_one, @function
add_one:
    leaq 1(%rdi), %rax
    ret
.size add_one, .-add_one

.section .note.GNU-stack,"",@progbits
```

Call it from a C file in the same app:

```c
extern long add_one(long value);

int main(void)
{
    return add_one(41) == 42 ? 0 : 1;
}
```

In the x86-64 System V ABI used by the cross-toolchain, this example receives
its first integer argument in `%rdi` and returns the result in `%rax`.
Preserve callee-saved registers when using them, keep stack alignment when
calling other functions, and avoid clobbering registers or machine state
without documenting the interface.

The SDK startup library provides the executable entry and calls the C
`main`. App assembly should normally export helper functions rather than
defining `_start`. Build examples with `make toolchain` followed by
`./sdk/scripts/build-example.sh EXAMPLE`; an app with an assembly file is
built just like one containing only C.

## NASM in low-level code

The kernel and boot path use NASM in places that require direct control over
CPU mode, registers, interrupts, or context switching. For example,
`boot/boot.S` is passed to NASM by the top-level Makefile despite the `.S`
suffix, while kernel files such as `kernel/arch/x86_64/interrupts.asm` use
NASM's `.asm` syntax. Do not copy a GAS app example into a NASM source file
or vice versa; follow the build rule for that exact source.

Low-level assembly is coupled to the linker script, kernel entry sequence,
interrupt ABI, and compiler options. Read the adjacent C/header code and
build rule before changing it. The normal `make` target checks the pinned
cross-toolchain and assembles the kernel.

## When to use assembly

Use C for ordinary app logic and device-independent code. Assembly is useful
for a small ABI boundary or operations that cannot be expressed safely in C.
Keep the interface narrow and test the combined C/assembly application; a
successful assembly step alone does not verify register preservation or
runtime behavior.

## A working SDK project

The SDK discovers both `main.c` and `math.S` below `src/`. The project
Makefile can remain the normal small SDK Makefile:

```make
APP_NAME := assembly-demo
APP_VERSION := 0.1.0
APP_SRC_DIR := $(CURDIR)/src
SDK_ROOT ?= /path/to/CreateOS/sdk

include $(SDK_ROOT)/app.mk
```

The C file declares the assembly symbol with a type that matches the
assembly function:

```c
#include <stdio.h>

extern long add_three(long value);

int main(void)
{
    long answer = add_three(39);
    if (answer != 42) {
        puts("assembly helper returned the wrong value");
        return 1;
    }
    puts("assembly helper returned 42");
    return 0;
}
```

`src/math.S`:

```asm
.text
.globl add_three
.type add_three, @function
add_three:
    leaq 3(%rdi), %rax
    ret
.size add_three, .-add_three

.section .note.GNU-stack,"",@progbits
```

Build it with the CreateOS cross-toolchain:

```sh
make toolchain
make -C /path/to/assembly-demo \
  SDK_ROOT="$PWD/sdk" \
  TOOLCHAIN_PREFIX="$PWD/build/toolchain/x86_64-elf"
```

The output is `assembly-demo.APPL` below the SDK's selected build directory.
To place it in a guest image, use the SDK install target or copy it into
`rootfs_maker/root/appcustom/`, then stage userland and rebuild the image.
Read [application development](app-development.md) before choosing the image
workflow.

## GAS syntax in more detail

GAS uses AT&T syntax for the SDK's `.S` files:

- registers begin with `%`, such as `%rax` and `%rdi`;
- immediate constants begin with `$`, such as `$1`;
- memory operands use `displacement(base,index,scale)`;
- most instructions write the source before the destination;
- instruction suffixes distinguish operand widths: `b`, `w`, `l`, and `q`.

For example, `movq %rdi, %rax` copies a 64-bit value from `%rdi` to `%rax`.
`movl $7, %eax` loads a 32-bit immediate. A write to a 32-bit general
register clears the upper half of the corresponding 64-bit register on
x86-64; account for that when mixing widths.

The `leaq` instruction calculates an address expression without loading
memory. In `leaq 8(%rdi,%rsi,4), %rax`, the computed value is
`%rdi + %rsi * 4 + 8`.

Typical directives:

| Directive | Purpose |
| --- | --- |
| `.text` | Place following code in the code section. |
| `.data` | Place initialized writable data. |
| `.bss` | Reserve zero-initialized storage. |
| `.globl name` | Export a symbol to other object files. |
| `.type name, @function` | Mark a symbol as a function for ELF tools. |
| `.size name, .-name` | Record the function's byte size. |
| `.section .note.GNU-stack,"",@progbits` | Mark that the object does not request an executable stack. |
| `.byte`, `.long`, `.quad` | Emit one-, four-, or eight-byte values. |

An uppercase `.S` source is preprocessed before assembly. Use this only when
you need a C preprocessor macro or conditional; keep regular assembly
readable as assembly rather than building complex logic into preprocessor
macros.

## Registers and the C calling convention

SDK applications use the x86-64 System V calling convention expected by the
cross compiler. For ordinary integer and pointer arguments:

| Argument | Register |
| --- | --- |
| 1 | `%rdi` |
| 2 | `%rsi` |
| 3 | `%rdx` |
| 4 | `%rcx` |
| 5 | `%r8` |
| 6 | `%r9` |

Integer and pointer return values are normally placed in `%rax`. Floating
point arguments and results use the ABI's XMM registers; do not write an
assembly interface for floating point until you have checked the compiler
ABI and the exact type being passed.

The caller-saved general registers are `%rax`, `%rcx`, `%rdx`, `%rsi`,
`%rdi`, `%r8` through `%r11`. A called function may overwrite them. If a
value must survive a call, save it in a callee-saved register or on the
stack.

The callee-saved general registers are `%rbx`, `%rbp`, and `%r12` through
`%r15`. If an assembly function changes one, it must restore the original
value before returning. `%rsp` must also be restored to the value expected
by the caller.

Do not assume a value survives across a call just because it did in one
debug build. The compiler may use caller-saved registers for temporary
values.

## Stack discipline

The System V ABI requires the stack to be 16-byte aligned immediately
before a `call` instruction. The `call` pushes an 8-byte return address, so
the callee normally begins with `%rsp` offset by eight bytes from a
16-byte boundary.

A leaf function that neither calls another function nor allocates stack
storage can often avoid modifying `%rsp` entirely. When a function needs a
stack frame, calculate its size so that the alignment is correct at every
call:

```asm
.text
.globl call_helper
.type call_helper, @function
call_helper:
    pushq %rbx
    # Entry has the return address on the stack. The push adjusts alignment.
    movq %rdi, %rbx
    call helper
    movq %rbx, %rax
    popq %rbx
    ret
.size call_helper, .-call_helper
```

This example's exact alignment depends on its entry convention and number
of stack adjustments; verify it rather than copying the comment as a
substitute for calculation. For a function that makes calls, write down
`%rsp mod 16` at entry and immediately before every call. Pushes and
subtractions change that value.

The CreateOS compiler flags disable the x86-64 red zone. Do not use memory
below `%rsp` as a hidden scratch area. The kernel has different interrupt
and context-switch constraints; never assume an app's stack rules are safe
for kernel code.

## Branches, loops, and flags

Compare values and branch on the flags set by the compare:

```asm
    cmpq $0, %rdi
    je .empty
    movq $1, %rax
    ret
.empty:
    xorq %rax, %rax
    ret
```

In AT&T syntax, `cmpq $0, %rdi` conceptually compares the value in `%rdi`
with zero. Conditional branches include `je`/`jz` for equal/zero,
`jne`/`jnz` for not equal, `jl` and `jg` for signed comparisons, and `jb`
and `ja` for unsigned comparisons.

Signed and unsigned branches are different. Select one based on the C type:
an `int` is normally tested with signed branches; a `uint64_t` is normally
tested with unsigned branches. Equality branches do not have that
signedness distinction.

A loop uses a label and a back edge:

```asm
    xorq %rax, %rax
.loop:
    cmpq %rsi, %rax
    jae .done
    incq %rax
    jmp .loop
.done:
```

This increments `%rax` until it reaches `%rsi`, and returns the final
counter in `%rax`. For real code, define a C prototype and a contract for
the input range and result. Test zero, one, and the largest supported input.

## Data and address access

Use `.rodata` for constant bytes and `.data` for writable initialized
storage. Avoid writable global state unless the API needs it.

```asm
.section .rodata
message:
    .asciz "hello from GAS"

.text
.globl message_address
.type message_address, @function
message_address:
    leaq message(%rip), %rax
    ret
.size message_address, .-message_address
```

This function returns the address of the string; the matching C prototype
is `extern const char *message_address(void);`.

The source tree's app link is static and compiled with `-fno-pic`. Do not
assume position-independent-code sequences or dynamic relocations are
available. Follow the static app linker script and existing symbols if you
need to share data across files.

## Calling a syscall from assembly

Normal app code should call the declared `sys_*` C wrappers in
`rootfs_maker/lib/user_syscall.h`; the wrappers in `syscalls.S` set the
CreateOS syscall number and execute `syscall`. This keeps syscall numbers,
argument registers, and return conventions in one project interface.

Do not copy a Linux syscall number or Linux-specific syscall recipe into a
CreateOS app. Similar register conventions do not make syscall numbers or
kernel semantics compatible. If you have a compelling reason to issue a
syscall instruction directly, read the matching kernel handler and wrapper,
write down the exact argument and error convention, and test invalid as
well as successful inputs.

## NASM boot and kernel assembly

NASM uses a different spelling and operand order:

```nasm
bits 64
global add_three
section .text
add_three:
    lea rax, [rdi + 3]
    ret
```

GAS example:

```asm
leaq 3(%rdi), %rax
```

NASM example:

```nasm
lea rax, [rdi + 3]
```

Do not mix the two forms in the same file. The top-level Makefile invokes
NASM for `boot/boot.S` and selected `.asm` files. The boot source establishes
the CPU state needed before entering C; interrupt and context-switch
assembly also has a kernel-specific ABI.

The boot entry begins in 32-bit mode, saves the Multiboot2 information
pointer, builds page tables, enables long mode, sets segment state, then
calls `kernel_main` in 64-bit mode. Any change to that sequence must stay
consistent with `linker.ld`, the GRUB Multiboot2 header, the C entry point,
and the processor-state assumptions of the kernel.

For kernel assembly:

1. Find the Makefile rule that assembles the file.
2. Read the adjacent C caller and header declarations.
3. Document input registers, output registers, and clobbers.
4. Preserve required registers and stack state.
5. Rebuild the kernel and test the behavior in QEMU.

## Debugging assembly

When a mixed C/assembly program fails:

1. Confirm that the symbol name is identical in the C declaration and
   assembly `.globl` directive.
2. Confirm argument types and register widths agree.
3. Confirm the return value is in the register expected by the C type.
4. Check every callee-saved register and `%rsp` restoration.
5. Check stack alignment before each call made by the assembly function.
6. Reduce the function to one return value and one test case.
7. Add boundary cases and a test that exercises the app in QEMU.

A successful assembler exit only proves that the syntax was accepted. It
does not prove that a register is preserved, a pointer is valid, or the
function follows the C calling convention.

## Exercises

1. Write an assembly function that returns its one integer argument.
2. Add a second argument and return their sum.
3. Write a function that returns 1 for zero and 0 otherwise.
4. Implement a loop that counts from zero up to a caller-supplied bound.
5. Call the function from C with negative, zero, and positive values.
6. Temporarily omit saving a callee-saved register and observe why the
   resulting bug can depend on compiler output.
7. Add a stack local to a function that calls C, then verify alignment.
8. Compare the app's GAS rule with the NASM rule for `boot/boot.S`.

For the SDK project pattern and image installation, see
[application development](app-development.md). For kernel entry and build
context, see the [kernel guide](../kernel/kernel.md) and
[build guide](../kernel/build-and-run.md).
