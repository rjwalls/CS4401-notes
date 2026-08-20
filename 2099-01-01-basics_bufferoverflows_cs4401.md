---
title: "Lecture Notes: Basics of Buffer Overflows (CS4401)"
date: 2099-01-01 02:00:00
categories: notes lecture
layout: post
challenges: stack0r-64 stack1r-64 stack2r-64 stack0r-guessdown stack2r-warp-drive
---

Over the first few lectures, we use the same basic memory-safety bug to build a
sequence of increasingly powerful exploits. Keeping the bug simple lets us
concentrate on the process of understanding a binary rather than guessing at an
exploit string.

The sequence moves through three targets:

1. change an adjacent integer to any nonzero value;
2. change an adjacent integer to one exact value; and
3. change a function pointer and redirect control flow.

The challenge binaries are x86-64 artifacts built in a pinned Docker authoring
environment. They run as ordinary shell challenges; students do not need to
start Docker containers to solve them. The expected solution is a repeatable
Python exploit script written with pwntools.

## `stack0`: Adjacent Data Corruption

### Lecture concept map

* attacker-controlled standard input and an oversized `read()`
* buffers, stack objects, and virtual addresses
* explicit structure layout and overwrite direction
* measuring offsets with source and GDB/GEF
* raw bytes versus typed values
* `p32()`, `flat()`, and `send()`

The first challenge uses an explicit structure:

```c
struct security_console {
    char input[64];
    volatile uint32_t security_mode;
};

int main(void) {
    struct security_console console = {
        .input = {0},
        .security_mode = 0
    };

    read(STDIN_FILENO, console.input, sizeof(console));
    if (console.security_mode != 0) {
        /* print the flag */
    }
}
```

The destination object is the 64-byte `input` array, but the program asks
`read()` for the 68-byte size of the entire structure. The function therefore
continues writing after the array ends and into `security_mode`.

That is a **buffer overflow**: the program writes beyond the bounds of a memory
object. C does not automatically stop the write when it reaches the end of the
array. What happens next depends on which object occupies the following
addresses.

Conceptually, the structure looks like this:

```text
Lower addresses                                Higher addresses
      -------------------------------------------------------->

      +-------------------------------+---------------+
      |          input[64]            | security_mode |
      +-------------------------------+---------------+
      ^                               ^
      read begins here                offset 64
```

C preserves the order of structure members, although it may insert padding for
alignment. The source gives us a strong model, but the compiled binary remains
the final authority. We can verify the offset in GDB/GEF:

```gdb
p &console.input
p &console.security_mode
x/80bx &console.input
```

Subtracting the two addresses gives the offset. Examining memory as bytes avoids
mistaking a debugger's typed interpretation for the underlying representation.

A small pwntools exploit can express the write directly:

```python
from pwn import ELF, context, flat, p32, process

context.binary = elf = ELF('./stack0-64')
payload = flat({64: p32(1)})

p = process(elf.path)
p.send(payload)
print(p.recvall().decode(errors='replace'))
```

`p32(1)` encodes a 32-bit integer in the target architecture's byte order.
`flat()` makes the offset visible in the exploit instead of hiding it inside a
long literal string. Because `read()` accepts raw bytes, `send()` transmits the
payload without appending an unrelated newline.

After this lecture, `stack0r-guessdown` asks you to transfer the same mental
model to a remote black-box problem whose exact layout changes at runtime.

## `stack1`: Exact Values and Environment Input

### Lecture concept map

* environment variables as process input
* `getenv()`, `strcpy()`, and C strings
* exact overwrites and cyclic offsets
* 32-bit integers and little-endian byte order
* Python `bytes`, `p32()`, and `flat()`
* constructing a child-process environment with pwntools

Stack 0 accepts any nonzero value. Stack 1 checks for one specific value:

```c
#define DEBUG_PASSWORD 0x45444f4d

struct generator_console {
    char input[64];
    volatile uint32_t debug_password;
};

const char *debug = getenv("DEBUG");
strcpy(console.input, debug);

if (console.debug_password == DEBUG_PASSWORD) {
    /* print the flag */
}
```

This time the attacker-controlled input is the `DEBUG` environment variable.
`getenv()` returns a pointer to its value, and `strcpy()` copies that
null-terminated string without knowing the size of `console.input`.

Environment variables are one of many possible process inputs. Others include
standard input, command-line arguments, files, network connections, and shared
memory. A useful recurring question is:

> What information in this process can I control, and where does the program
> copy or use it?

### Little endian

x86-64 uses little-endian byte order. A 32-bit value written as:

```text
0x12345678
```

is stored from lower to higher addresses as:

```text
78 56 34 12
```

The target value has a convenient representation:

```python
p32(0x45444f4d) == b'MODE'
```

The integer is written as `0x45444f4d`, but the bytes appear in memory in the
reverse order. GDB can display the same four bytes as an integer, individual
bytes, or characters. The memory has not changed; only our interpretation has.

### Measure instead of guessing

The program prints the overwritten debug value, which makes a cyclic pattern a
useful measuring tool:

```python
from pwn import cyclic

env = {b'DEBUG': cyclic(128)}
```

After observing the printed value, `cyclic_find()` identifies where those bytes
occurred in the pattern. Source inspection or subtracting the member addresses
in GDB should independently confirm the 64-byte offset.

The final exploit constructs the environment directly:

```python
from pwn import ELF, context, flat, p32, process

context.binary = elf = ELF('./stack1-64')
payload = flat({64: p32(0x45444f4d)})

p = process(elf.path, env={b'DEBUG': payload})
print(p.recvall().decode(errors='replace'))
```

This keeps input construction in the exploit script and makes the distinction
between Python text and bytes explicit. Prefer byte-oriented exploit code over
shell quoting tricks once a payload contains packed values.

## `stack2`: Function Pointers and Control Flow

### Lecture concept map

* pointers, function pointers, and code pointers
* indirect calls, `rip`, and control-flow hijacking
* 64-bit pointers and `p64()`
* `file`, `checksec`, and binary mitigations
* non-PIE executables and stable code addresses
* ELF symbols, `flat()`, and exploit verification

Stack 2 places a function pointer after the input buffer:

```c
struct navigation_console {
    char input[64];
    void (*handler)(void);
};

if (console.handler != NULL) {
    console.handler();
} else {
    diagnostics();
}
```

A function pointer is data that contains the address of executable code. When
the program performs an indirect call through that pointer, the value influences
the instruction pointer, `rip`. Replacing the empty handler with the address of
`recalibration()` changes which instructions the processor executes. This is
**control-flow hijacking**.

On x86-64, pointers are eight bytes. Use `p64()` or let `flat()` pack values
using an `amd64` `context.binary` instead of manually reversing an address.

### Check whether an address is stable

Before placing an address in an exploit, inspect the binary:

```bash
file ./stack2-64
checksec ./stack2-64
```

The CS4401 Stack 2 executable is deliberately non-PIE. ASLR may randomize other
process regions, but addresses in this executable's text mapping remain stable
between runs. The ELF symbol value for `recalibration` is therefore also its
runtime address:

```python
from pwn import ELF

elf = ELF('./stack2-64')
assert not elf.pie
recalibration = elf.symbols['recalibration']
```

The complete overwrite follows the same measured-layout process used before:

```python
from pwn import ELF, context, flat, p64, process

context.binary = elf = ELF('./stack2-64')
target = elf.symbols['recalibration']
payload = flat({64: p64(target)})

p = process(elf.path)
p.send(payload)
print(p.recvall().decode(errors='replace'))
```

Run the script under GDB/GEF and stop after the vulnerable `read()`. Confirm the
function-pointer value in memory, step to the indirect call, and watch control
move to `recalibration`. An exploit is a model of the program that you can test,
not a collection of guessed constants.

After Stack 2, complete the Ghidra Fundamentals mini-lab with this
source-provided binary. Then use the same analysis process on
`stack2r-warp-drive`, where the source is withheld.

## Practical Exploit-Development Workflow

For these early challenges:

1. Read the source when it is provided and identify attacker-controlled input.
2. Find the unsafe operation and the state you want to corrupt.
3. Run `file` and `checksec` before assuming anything about the binary.
4. Draw the relevant objects in address order and measure their offsets.
5. Construct explicit bytes with pwntools helpers such as `p32()`, `p64()`, and
   `flat()`.
6. Run the actual exploit script under GDB/GEF and inspect the resulting memory.
7. Verify the target value, the instruction that consumes it, and every address
   assumption.

Develop on the x86-64 course shell server so the architecture and tools match
the challenge environment. You can keep an exploit in your home directory and
run it while your working directory is the deployed challenge directory:

```bash
cd /problems/CHALLENGE_INSTANCE/
python3 ~/exploits/stack2.py
```

Relative paths opened by the exploit are resolved from the current working
directory. This is especially important for binaries that expect `flag.txt`, a
bundled loader, or other files beside the executable.

## Summary

These challenges progress from adjacent data corruption to exact byte-oriented
writes and finally to control-flow hijacking. The transferable skills are:

* identifying attacker-controlled process input;
* reasoning about object layout and overwrite direction;
* treating memory as bytes and types as interpretations;
* measuring offsets with GDB/GEF instead of guessing;
* encoding integers and pointers with pwntools; and
* checking whether an address remains valid across executions.

The next stack sequence replaces the convenient adjacent function pointer with
a code pointer that ordinary function calls place on the stack: the saved return
address.
