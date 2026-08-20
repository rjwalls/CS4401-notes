---
title: "Lecture Notes: Basics of Buffer Overflows (CS557)"
date: 2020-01-01 02:00:00
categories: notes lecture
layout: post
challenges: stack0g-64 stack1g-64 stack2g-64 stack0g-guessdown stack2g-warp-drive
---

Over the first few lectures, we are going to jump straight into binary exploitation. We will start with a very simple bug---a stack-based buffer overflow caused by an oversized read---and, somewhat intentionally, keep exploiting essentially that same bug for several challenges.

Why keep using the same bug? Because the buffer overflow itself is not the hard part. I want to keep that part simple while we introduce the other concepts that you need in order to exploit binaries:

* How is memory organized?
* What exactly is a buffer overflow?
* How do we figure out where objects actually live in memory?
* What is endianness, and why does it matter?
* What is a code pointer?
* What does it mean to hijack a program's control flow?
* How do we use GDB, GEF, and pwntools to answer these questions instead of guessing?

The targets will gradually become more interesting. Roughly speaking, we will progress from

1. changing an integer while preserving adjacent integrity-critical state,
2. changing an integer to an *exact value*, and
3. changing a **function pointer**, which lets us begin manipulating the program's control flow.

Later, we will take another step and target a particularly important code pointer: the **saved return address**. But we have a fair amount of groundwork to cover before we get there.

## Course objectives and solver path

These challenges use fixed, Docker-built x86-64 binaries so a shell-server upgrade
does not silently change offsets, symbols, libc behavior, or mitigations. Docker
is only the authoring environment; these are not Docker runtime challenges.

1. `stack0g-64`: change the target while preserving the adjacent integrity
   field.
2. `stack0g-guessdown`: infer a changing black-box layout and explain why
   a packed spray is robust across its possible aligned offsets.
3. `stack1g-64`: diagnose transformations across the shell, environment,
   Python bytes, and C strings; construct the newline-containing value directly
   in pwntools.
4. `stack2g-64`: distinguish ELF offsets from runtime addresses, recover a
   PIE base from a leak, and derive the recalibration function in the same
   process.
5. Complete the Ghidra Fundamentals mini-lab, then independently recover and
   exploit `stack2g-warp-drive` under ASLR.

The expected deliverable is a repeatable Python exploit script, not a one-off
terminal command.

<!-- The crew manifest says nobody is in this panel. The manifest is wrong.
You looked somewhere you were supposed to look, so take this before they notice:
DoTheRequiredReading -->

## Getting Started with `stack0`

### Lecture concept map

* attacker-controlled standard input and an oversized `read()`
* stack objects and explicit structure layout
* overwrite direction and measured offsets
* bounded corruption and preserved integrity-critical state
* raw memory inspection with GDB/GEF
* `p32()`, `flat()`, and exact payload length
* `send()` versus `sendline()`

Consider the basic structure of the first CS557 challenge:

```c
#include <stdint.h>
#include <unistd.h>

struct security_console {
    char input[64];
    volatile uint32_t security_mode;
    volatile uint32_t integrity;
};

int main(void) {
    struct security_console console = {
        .input = {0},
        .security_mode = 0,
        .integrity = 0x434c4157
    };

    read(STDIN_FILENO, console.input, sizeof(console));
    if (console.security_mode != 0 && console.integrity == 0x434c4157) {
        /* print the flag */
    }
}
```

Our goal is to convince this program to print the contents of `flag.txt`.

There is an obvious problem: `security_mode` starts at zero, the program never assigns another value to it, and the flag is only printed if `security_mode != 0`. At the same time, the adjacent `integrity` field must retain its original value.

Under the program's intended execution, we lose.

Fortunately, this is a binary exploitation course. We are not limiting ourselves to the program's intended execution.

### What does it mean to exploit a binary?

For our purposes, **exploiting a binary** means taking advantage of some flaw in a program to make the program behave in a way that its programmer did not intend.

For `stack0g-64`, the relevant intended behavior is:

```c
security_mode = 0;
integrity = 0x434c4157;
...
if (security_mode != 0 && integrity == 0x434c4157) {
    ...
}
```

The programmer clearly expects `security_mode` to remain zero. Our job is to make that assumption false without violating the integrity check. This is a small but important distinction: successful corruption is not necessarily unlimited corruption.

That gives us a useful way to think about an exploit. We generally need at least two things:

1. **A bug** that gives us some useful unintended behavior.
2. **Attacker-controlled input** that lets us trigger that bug.

Let's look at the input first.

## Supplying Input to a Program

Attackers can potentially influence programs in many different ways.

Input might arrive through

* standard input,
* command-line arguments,
* environment variables,
* files,
* network connections,
* shared memory,
* hardware devices,
* or any number of other interfaces.

One useful habit in binary exploitation is to ask:

> **What information in this process do I control?**

In `stack0`, the most obvious answer is standard input because of this line:

```c
read(STDIN_FILENO, console.input, sizeof(console));
```

`read()` reads from **standard input**, or `stdin`. If you execute the program from a terminal and type some characters, those characters are being provided through standard input.

Later challenges will force us to think about other input mechanisms. For example, environment variables matter in `stack1`.

For now, though, this oversized `read()` gives us exactly the interface we need.

## The Bug: A Size Mismatch

The destination is the 64-byte `input` member, but the requested read length is
the 72-byte size of the entire structure. That explicitly permits bytes to cross
from the input buffer into both adjacent fields. The bug gives us permission to
write too much; the integrity check gives us a reason to stop after exactly 68
bytes.

The infamously unsafe `gets()` function illustrates the same underlying lesson
and still appears in older examples and other challenge code. Its interface is
worth recognizing:

If you are unfamiliar with a C library function, one of the first things you should do is read its documentation.

On Linux, that frequently means using the manual pages:

```bash
man gets
```

This is also an important general lesson for the course: **look things up**. Do not rely on a fuzzy memory of what a function probably does when the exact semantics matter to your exploit.

`gets()` reads characters from standard input and copies them into the buffer provided by its argument. It stops when it reaches a newline or end-of-file.

There is a fairly substantial problem, though:

```c
gets(buffer);
```

Notice what is *not* passed to `gets()`.

There is no buffer size.

Suppose `buffer` is 64 bytes long. How is `gets()` supposed to know that?

It doesn't.

If we provide 10 bytes, `gets()` can write 10 bytes.

If we provide 64 bytes, it can write 64 bytes.

If we provide 100 bytes, it will continue writing those bytes even though the `buffer` object itself ended a long time ago.

That is our bug.

### Buffer overflows

A **buffer overflow** occurs when a program writes beyond the bounds of a memory object.

For example, imagine we have a 4-byte buffer beginning at address `0x1000`:

```text
Lower addresses                              Higher addresses
       ─────────────────────────────────────────────────►

        0x1000    0x1001    0x1002    0x1003
       +---------+---------+---------+---------+
buffer |         |         |         |         |
       +---------+---------+---------+---------+
```

Writing four bytes is fine.

Writing a fifth byte means writing to `0x1004`, which does **not** belong to `buffer`.

C does not automatically stop us from doing that.

What happens next depends entirely on what happens to live at `0x1004`.

And that question---**what is next to my buffer in memory?**---is going to be one of the most important questions we ask throughout this course.

### A small but important detail about `gets()`

There is one piece of `gets()` behavior that matters when we start counting bytes precisely.

If you type:

```text
AAAA
```

and press Enter, the terminal supplies a newline. `gets()` uses that newline to determine where the line ends, but **does not store the newline in the destination buffer**.

Instead, the resulting C string is:

```text
'A' 'A' 'A' 'A' '\0'
```

That final `\0` is the **null terminator**.

This distinction becomes important when we are trying to overwrite an object at an exact offset.

If another variable begins exactly 132 bytes after the start of our buffer, then sending 132 `a` bytes causes the null terminator to be written at that next location. If the target is an integer that already contains zero, we have accomplished precisely nothing.

We would need to send at least one additional nonzero byte:

```python
p.sendline(b'a' * 133)
```

These sorts of one-byte details matter. Being off by one byte is very often the difference between "exploit works" and "exploit crashes."

The current Stack 0 challenge uses `read()` and does not append this terminator.
The detail still matters for functions that create C strings, including the
`strcpy()` path in Stack 1.

## A Quick Review of Memory

To understand what we can overwrite, we need a model of where program data lives.

When a program executes, the operating system creates a **process**. Among other things, each process is given its own **virtual address space**.

A useful way to visualize a virtual address space is as a gigantic array of bytes:

```text
address 0
   |
   v
+------+------ +------+------ +------ ...
| byte | byte  | byte | byte  | byte
+------+------ +------+------ +------ ...
   0      1       2      3       4
```

Every byte has an address.

The program generally deals in these **virtual addresses** rather than directly referring to physical locations in RAM.

That abstraction makes programming substantially easier and also helps isolate processes from one another.

### Important regions

Different parts of the virtual address space are used for different purposes. For now, the three regions we care about most are:

* **Text / executable mappings:** contain the machine instructions that make up the program.
* **Heap:** contains dynamically allocated memory, such as memory returned by `malloc()`.
* **Stack:** contains information associated with function execution, including local variables.

We will spend plenty of time on all three.

For `stack0`, the important one is the stack because this structure is a local variable:

```c
struct security_console console = {
    .input = {0},
    .security_mode = 0,
    .integrity = 0x434c4157
};
```

Local variables normally live in the function's **stack frame**.

We will give stack frames a much more careful treatment when we get to return addresses. For now, we mostly care that `console.input`, `console.security_mode`, and `console.integrity` are sitting in one structure in stack memory.

## Exploiting `stack0`

We now have all the pieces we need.

The oversized `read()` starts writing at the beginning of `console.input` and
continues toward higher addresses.

Because C preserves structure-member order, `console.security_mode` follows the
input array, followed by `console.integrity`. A sufficiently long input runs off
the end of the array and begins overwriting those fields.

Conceptually, perhaps memory looks like this:

```text
◄──────────────── Lower addresses       Higher addresses ────────────────►

+-------------------------------+---------------+-----------+
|         console.input         | security_mode | integrity |
+-------------------------------+---------------+-----------+
^
read() starts writing here ──────────────────────►
```

If we write 64 bytes of padding followed by one packed 32-bit value:

```text
+-------------------------------+---------------+-----------+
| A A A A A A A A A A A A A A | 01 00 00 00   | 57 41 ... |
+-------------------------------+---------------+-----------+
                                  ^               ^
                                  changed         preserved
```

And suddenly:

```c
if (console.security_mode != 0 && console.integrity == 0x434c4157)
```

is true.

That is our first exploit. The useful payload ends immediately after the
four-byte `security_mode` value. Additional bytes would begin corrupting the
integrity field and turn a successful overwrite into a failed repair.

The attack itself is simple:

> **Use a buffer overflow to change useful adjacent state while preserving the state the program still verifies.**

That basic description will continue to apply to the next few challenges. We will just keep choosing more interesting things to overwrite.

## Use Explicit Layouts, Then Verify Them

There is an important catch.

For separate local variables such as:

```c
int security_mode;
char buffer[64];
```

you should **not** conclude from declaration order that the compiler must put
`security_mode` immediately before or after `buffer`.

The compiler determines the actual layout.

It could look like this:

```text
+----------------------+------------+
|       buffer         | security_mode |
+----------------------+------------+
```

or:

```text
+------------+----------------------+
| security_mode |       buffer         |
+------------+----------------------+
```

or:

```text
+----------------------+-----+----------------+------------+
|       buffer         | ??? |      ???       | security_mode |
+----------------------+-----+----------------+------------+
```

There may be padding. There may be other saved values. Compiler choices may
change between builds. Our current challenges deliberately place the relevant
objects in an explicit structure, which guarantees member order, but the
compiler may still insert padding for alignment. The binary remains the final
authority for the exact offsets.

The source code tells us what the programmer wrote.

The **binary tells us what will actually execute**.

That means we need tools.

## Use GDB to Measure, Not Guess

One of the habits I want you to develop very early in this course is:

> **Do not guess your way through an exploit.**

Suppose your exploit almost works.

It is tempting to say:

> Maybe I need 72 `a`s instead of 71.

Then:

> Hmm. Maybe 73?

Then:

> Maybe I should put quotes around this number?

This style of debugging works surprisingly often in introductory programming courses because small modifications can move you gradually toward the right answer.

Binary exploitation is different.

If the correct address is

```text
0x5555555547aa
```

and you use

```text
0x5555555547ab
```

your program does not become "almost correct."

It may simply crash.

You get essentially no useful gradient telling you whether your random modification moved closer to the solution.

So when something does not work:

1. Draw the relevant memory.
2. Determine where each object begins and ends.
3. Look at the actual bytes.
4. Step through instructions.
5. Check every assumption.

That sounds slower than guessing.

It is much faster.

### Starting with pwntools

For these challenges, I recommend writing a pwntools script from the beginning rather than constructing increasingly complicated shell one-liners.

A minimal starting point might look something like this:

```python
from pwn import *

elf = ELF('./challenge/stack0-64')
context.binary = elf
context.log_level = 'debug'

p = gdb.debug(elf.path, cwd='./challenge', gdbscript='''
    break source.c:37
    continue
''')

payload = flat({64: p32(1)})
assert len(payload) == 68
p.send(payload)

p.recvall()
p.wait()
```

There are several useful things going on here.

First:

```python
elf = ELF('./stack0-64')
context.binary = elf
```

tells pwntools which binary and architecture we are working with.

`ELF()` will also display the results of `checksec`, which gives us an early look at the defenses enabled for the binary.

Next:

```python
context.log_level = 'debug'
```

gives us much more detailed output. In particular, pwntools will show the bytes being transmitted and received. That is enormously useful when the distinction between

```text
"41414141"
```

and

```text
41 41 41 41
```

matters.

The explicit length check matters in the graduate challenge. `send()` transmits
exactly the bytes in `payload`; `sendline()` would append a newline and begin
overwriting the integrity field.

Finally:

```python
p = gdb.debug(...)
```

runs the program under GDB/GEF so that we can inspect the process while our *actual exploit script* interacts with it.

That is generally much more useful than having one test input in GDB, a different Python one-liner outside GDB, and a third version embedded in an exploit script.

### Looking at memory

Once we have stopped the program after the vulnerable `read()`, commands like these become useful:

```gdb
info locals
```

```gdb
p &console.input
```

```gdb
x/s &console.input
```

```gdb
x/96bx &console.input
```

The last command is especially important.

`x/s` asks GDB to interpret memory as a C string. Sometimes that is exactly what we want.

But ultimately memory contains **bytes**. If we are trying to determine precisely what our exploit changed, looking at the raw bytes with `x/...bx` often gives us a much clearer answer.

GEF also gives us commands such as:

```gdb
context
```

```gdb
vmmap
```

```gdb
xinfo ADDRESS
```

and:

```gdb
checksec
```

We will use these constantly.

## Looking at the Disassembly

Eventually we also need to become comfortable answering questions from the machine instructions themselves.

For example, imagine we see something like this near the call to `read()`:

```asm
mov    edx,0x48
lea    rax,[rbp-0x50]
mov    rsi,rax
mov    edi,0x0
call   read@plt
```

The x86-64 System V calling convention places the first three integer or
pointer arguments in `rdi`, `rsi`, and `rdx`.

`read()` takes a file descriptor, destination address, and byte count.

Therefore, immediately before the call:

```text
RDI = 0 (standard input)
RSI = address of console.input
RDX = 72 bytes
```

That is incredibly useful.

Rather than trying to reverse-engineer every instruction in `main`, we can often go straight to the vulnerable function call, place a breakpoint nearby, and examine the argument register:

```gdb
p/x $rsi
```

or:

```gdb
x/96bx $rsi
```

This is a recurring exploit-development strategy:

> **Use what you already know about the function interface and calling convention to find the information you care about.**

You do not have to understand every instruction in a binary before you can exploit it.

You do, however, need to understand the instructions that matter to your attack.

## `stack1`: Exact Values and Endianness

### Lecture concept map

* environment variables as process input
* `getenv()`, `strcpy()`, and C-string boundaries
* exact overwrites instead of merely nonzero values
* 32-bit integers, little endian, and `p32()`
* Python `bytes` versus text
* shell, environment, Python, and C data boundaries
* direct pwntools `env` construction and byte verification

In `stack0`, the target value itself is flexible.

We do not particularly care what value we place in `security_mode`. We only need:

```c
security_mode != 0
```

Any nonzero 32-bit value is acceptable as long as the following integrity field
survives. Stack 1 makes the value itself exact.

The CS557 challenge uses the newline-containing `0x0d0a0d0a` value:

```c
if (console.debug_password == 0x0d0a0d0a) {
    ...
}
```

Now sending:

```python
b'a' * 100
```

is not sufficient.

We need the exact bytes corresponding to:

```text
0x0d0a0d0a
```

And that means we need to understand two things:

1. the size of the object, and
2. **endianness**.

### Little endian

x86 and x86-64 systems use **little-endian** byte ordering.

Suppose we want to store the 32-bit value:

```text
0x12345678
```

As humans, we usually write the number with the most significant byte first:

```text
12 34 56 78
```

But in memory on a little-endian machine, the bytes are stored:

```text
Lower address                     Higher address
     ─────────────────────────────────────►

     78        56        34        12
```

This distinction causes a lot of confusion early in the course.

One additional source of confusion is that GDB may display the four bytes as the integer:

```gdb
0x12345678
```

even though examining them individually gives:

```gdb
0x78 0x56 0x34 0x12
```

Neither view is wrong.

You told GDB to interpret the same bytes differently.

And that brings us to an important point:

> **Memory contains bytes. Types are interpretations that we place on those bytes.**

Four bytes might be interpreted as

* a 32-bit integer,
* four ASCII characters,
* part of an instruction,
* part of a pointer,
* or something else entirely.

The bytes themselves do not know what type they are.

### Let pwntools handle the packing

We could manually reverse every integer into little-endian byte order.

Please do not do that unless you have a good reason.

Pwntools already gives us helpers for this:

```python
p32(0x0d0a0d0a)
```

packs a 32-bit integer.

Similarly:

```python
p64(0xdeadbeef)
```

packs a 64-bit integer.

This lets the exploit express what we actually mean:

```python
payload = b'a' * offset
payload += p32(0x0d0a0d0a)
```

instead of manually constructing an error-prone sequence of escaped bytes.

### Environment variables as input

`stack1` also provides a useful reminder that stdin is not the only way to control a process.

The challenge obtains `DEBUG` from the process environment and then copies it
without checking its length:

```c
const char *debug = getenv("DEBUG");
strcpy(console.input, debug);
```

Environment values reach C as null-terminated byte strings. `strcpy()` copies
those bytes, followed by a null terminator, until it encounters the end of the
source string. Environment values therefore cannot contain an embedded null
byte, but they can contain line-feed and carriage-return bytes.

If the vulnerable program reads from an environment variable, pwntools can set that environment for us:

```python
from pwn import *

p = process(
    './stack1-64',
    env={
        b"DEBUG": b"a" * 64 + p32(0x0d0a0d0a)
    }
)

p.interactive()
```

This direct byte-valued mapping is the intended and most dependable approach.
It passes the payload to the child process without asking a shell to parse or
store it first.

Be precise about what the shell does, however. Shell command substitution strips
**trailing** newline bytes. The packed value in this challenge is:

```python
p32(0x0d0a0d0a) == b'\x0a\x0d\x0a\x0d'
```

Because this particular sequence ends in a carriage return, carefully quoted
command substitution can preserve it. That does not make the shell a good
general-purpose binary transport. A payload that ends in a newline would be
changed, shell quoting and substitution rules remain context dependent, and
embedded null bytes cannot pass through either an environment variable or a C
string. Build and inspect the bytes in the exploit script so that each boundary
is explicit.

Again, the high-level question is:

> What data in this process can I control, and where does the program copy that data?

That question generalizes much better than memorizing a particular exploit string.

## `stack2`: From Data Corruption to Control-Flow Hijacking

### Lecture concept map

* function pointers, code pointers, indirect calls, and `rip`
* 64-bit pointers and `p64()`
* `file`, `checksec`, and ELF defenses
* PIE, ASLR, ELF-relative offsets, and runtime addresses
* parsing an information leak from the running process
* `PIE base = runtime leak - ELF symbol offset`
* rebasing a pwntools `ELF` and deriving `recalibration`
* using the leak and overwrite in the same process

So far, our target has been an integer.

That is useful, but it is somewhat limited.

The next step is to overwrite something substantially more powerful: a **function pointer**.

A function pointer is a C variable that contains the address of a function.

For example:

```c
void (*fp)();
```

might eventually contain the address of:

```c
recalibration()
```

and the program can then indirectly call the function through `fp`.

I like to place function pointers into a larger conceptual category that we will use throughout this course: **code pointers**.

## Code Pointers

A **code pointer** is a value in memory that contains the address of executable code and will eventually be used to determine what instructions the processor executes.

On x86-64, pointers are 8 bytes.

Examples of code pointers that we will encounter include:

* function pointers,
* saved return addresses,
* entries associated with dynamic linking,
* entries in C++ virtual-function tables,
* and various other control-flow data structures.

Why are these so interesting?

Because eventually the address stored in a code pointer influences the **instruction pointer**.

On x86-64, the instruction pointer register is `rip`.

`rip` identifies the next machine instruction the processor is going to execute.

You can watch it change as you single-step through a program in GDB.

If we can overwrite a code pointer with an address of our choosing, and the program later uses that corrupted pointer, we can potentially control what gets loaded into `rip`.

That gives us **control-flow hijacking**.

### Control-flow hijacking

**Control-flow hijacking** means making a program execute a different sequence of instructions than it was supposed to execute.

Suppose the program intends to call:

```text
diagnostics
```

but we corrupt a function pointer so that it instead points to:

```text
recalibration
```

Then the indirect call eventually causes:

```text
RIP = address of recalibration
```

We have changed the program's control flow.

This is a significant step up from changing an integer.

And it is the basic idea behind many of the attacks we will study for the rest of the course.

## Finding the Target Address

To overwrite the function pointer with the address of `recalibration`, we first
need to know the runtime address of `recalibration`.

If the address is fixed, GDB may make this very easy:

```gdb
p &recalibration
```

or:

```gdb
info address recalibration
```

But before blindly copying an address into an exploit, we should ask another question:

> **Is this address actually stable between executions?**

This is where two terms you will see repeatedly begin to matter:

* **PIE**
* **ASLR**

You can get a quick look at the binary's protections with:

```bash
checksec ./stack2-64
```

or from inside GEF:

```gdb
checksec
```

For the CS557 Stack 2 binary, you should see output resembling:

```text
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    No canary found
NX:       NX enabled
PIE:      PIE enabled
```

We will discuss all of these defenses in more detail later.

For now, concentrate on PIE.

## PIE and ASLR: The Short Version

**Address Space Layout Randomization (ASLR)** changes where important regions of a process are placed in virtual memory.

For example, on one execution our program might be loaded here:

```text
0x555555554000
```

and on another execution somewhere else.

This makes exploitation more difficult because attackers frequently need to know addresses.

**Position Independent Executable (PIE)** support allows the executable itself to be relocated in this way.

If PIE is disabled, the program's executable code normally remains at fixed addresses even when ASLR randomizes other regions such as the stack and shared libraries.

If PIE is enabled, addresses shown by static-analysis tools can instead look like small offsets:

```text
recalibration = 0x7aa
```

That does **not** necessarily mean that `recalibration()` is really executing at virtual address `0x7aa`.

Once the process is running, you might instead find something like:

```text
0x5555555547aa
```

The useful observation is that the offset inside the executable remains the same:

```text
base + 0x7aa
```

That is a consequence of the coarse-grained randomization we will discuss in our ASLR notes.

`stack2g-64` provides the runtime address of `diagnostics` as an **information
leak** from the current process. Once we have one known runtime address from the
executable, fixed symbol offsets let us calculate the PIE base and the runtime
address of `recalibration`.

The important point for now is not to memorize an ASLR bypass.

It is to start asking:

> **Where did this address come from, and is it still valid for the process I am exploiting?**

## Developing the Exploit Systematically

The graduate challenge gives us everything needed to perform that calculation
in a repeatable exploit:

```python
from pwn import ELF, context, flat, p64, process

context.binary = elf = ELF('./stack2-64')
diagnostics_offset = elf.symbols['diagnostics']

p = process(elf.path)
p.recvuntil(b'Diagnostic beacon: ')
diagnostics_runtime = int(p.recvline().strip(), 16)

pie_base = diagnostics_runtime - diagnostics_offset
assert pie_base & 0xfff == 0
elf.address = pie_base

recalibration_runtime = elf.symbols['recalibration']
payload = flat({64: p64(recalibration_runtime)})
p.send(payload)
print(p.recvall().decode(errors='replace'))
```

Save the static `diagnostics` offset before assigning `elf.address`. Once the
pwntools ELF object is rebased, symbol lookups return runtime addresses. The
page-alignment assertion is a useful sanity check: executable mappings begin on
page boundaries, so a correctly recovered base should end in three zero
hexadecimal digits.

Most importantly, do not let the process that produced the leak exit before you
send the overwrite. A new execution receives a new PIE base under ASLR. The
leak and the derived target are valid for the process that emitted that leak.

### Question 1: What is the offset?

Where does the destination buffer begin?

Where does the object we want to corrupt begin?

Subtract those addresses.

Draw it:

```text
Lower addresses                                        Higher addresses
      ─────────────────────────────────────────────────────────────►

      +-----------------------------+----------+------------------+
      |          buffer             | padding  |      target      |
      +-----------------------------+----------+------------------+
      ^                                        ^
      |                                        |
      buffer address                           target address

      offset = target - buffer
```

Then verify the result in GDB.

### Question 2: What bytes belong at the target?

What is the target's type?

If it is a 32-bit integer:

```python
p32(value)
```

If it is a 64-bit address:

```python
p64(address)
```

or let `flat()` use the architecture information from:

```python
context.binary = elf
```

### Question 3: Did our payload actually produce that memory state?

Run it in GDB.

Break after the vulnerable copy.

Examine the bytes.

Do not assume.

For example:

```gdb
x/128bx &buffer
```

or:

```gdb
x/gx ADDRESS
```

Check whether the target now contains exactly what you think it contains.

If not, fix your model before changing random values in your script.

## A Note on Controlled Environments and Working Directories

There is one practical detail that causes a surprising amount of confusion.

Usually I recommend developing your exploit in a **controlled environment** first.

In this course, that usually means the course shell server, not necessarily your laptop.

The reason is partly mundane but important: many of your laptops are ARM machines, while the challenge binaries are built for x86-64. We want an environment that mirrors the target binary's architecture closely enough, but still gives us the knobs turned in our favor. We can run the program under GDB/GEF, use pwntools, control standard input, command-line arguments, environment variables, files, and inspect memory while we work.

So, when possible, copy the binary into your own working area on the course shell server, make a fake `flag.txt`, and develop the exploit where you have control.

Once the exploit works, you may need to execute it against the actual challenge binary in the protected challenge directory.

You do **not** need to copy your exploit script into that directory.

Suppose your exploit lives here:

```text
~/exploits/stack2.py
```

and the real challenge is here:

```text
/problems/stack2g-64_INSTANCE_ID/
```

You can do:

```bash
cd /problems/stack2g-64_INSTANCE_ID/
python3 ~/exploits/stack2.py
```

If the script opens:

```python
ELF('./stack2-64')
```

then `./stack2-64` is resolved relative to the **current working directory**, not relative to the directory containing your Python script.

This becomes even more important later because the execution environment can affect memory layout, especially addresses on the stack.

## A Practical Exploit-Development Workflow

For these early challenges, I recommend roughly the following process.

### 1. Read the source, if you have it

Identify:

* attacker-controlled input,
* unsafe operations on that input,
* and values that might make useful corruption targets.

But remember: source code is only our starting point.

### 2. Run `file` and `checksec`

For example:

```bash
file ./stack2-64
checksec ./stack2-64
```

Know whether you are dealing with a 32- or 64-bit binary and start getting used to checking the enabled defenses.

### 3. Open it in GDB/GEF

Look at:

```gdb
disass main
```

Find the vulnerable call.

Use the calling convention to understand the arguments.

### 4. Draw the relevant memory

Actually draw the boxes.

Where is the buffer?

Where is the target?

Which direction does the overwrite move?

How many bytes separate them?

What size is each object?

### 5. Write the exploit in pwntools

Prefer explicit byte-oriented code:

```python
b'A' * offset
```

and helpers such as:

```python
p32(...)
p64(...)
flat(...)
```

over clever shell one-liners.

### 6. Run the exploit under GDB

Inspect the actual memory state produced by your script.

Step through the instructions that consume the corrupted value.

### 7. Verify every assumption

If it does not work, ask:

* Was my offset correct?
* Did I pack the value using the correct size?
* Is the byte order correct?
* Did I accidentally send ASCII characters instead of raw bytes?
* Is the address from this same process?
* Is PIE enabled?
* Is ASLR relevant?
* Did the program actually reach the instruction that uses the value I corrupted?

Those questions give you information.

Changing `72` to `73` because "maybe that works" generally does not.

## Related Challenges

This note is most directly useful for:

* `stack0g-64`: bounded adjacent-data corruption that preserves a neighboring
  integrity field.
* `stack1g-64`: an exact environment-variable overwrite built from explicit
  bytes.
* `stack2g-64`: function-pointer control-flow hijacking with leak-assisted PIE
  rebasing.

It also sets up:

* `stack0g-guessdown`: the same early questions in a remote black-box setting
  with a runtime-sized layout and timeout.
* `stack2g-warp-drive`: source-withheld Ghidra analysis followed by
  leak-assisted PIE rebasing.

The next set of notes on stack frames and return addresses will continue with
the saved-return-address challenges after that sequence has been updated.

## What Comes Next?

At this point, we have taken one simple bug and used it in increasingly powerful ways.

First:

```text
buffer overflow
      |
      v
change an integer to something nonzero
```

Then:

```text
buffer overflow
      |
      v
write an exact sequence of bytes
```

Then:

```text
buffer overflow
      |
      v
corrupt a function pointer
      |
      v
control RIP
      |
      v
control-flow hijacking
```

The natural next question is:

> What if the program does not conveniently put a function pointer immediately after our buffer?

Fortunately for us, function execution itself requires code pointers.

When one function calls another, the program eventually needs to know where execution should continue when that function returns.

That address---the **saved return address**---is stored on the stack.

And unlike our convenient function pointer in `stack2`, saved return addresses occur naturally in ordinary programs all the time.

Understanding how they get there requires us to look much more carefully at

* stack frames,
* the function prologue,
* the function epilogue,
* `call`,
* `ret`,
* `rsp`,
* and `rbp`.

That is where we will go next.

## Summary

The most important idea from these first challenges is not a particular exploit string.

It is the general process.

A buffer overflow lets a program write outside the bounds of a memory object. If useful data lives in the path of that overflow, we may be able to corrupt it.

The usefulness of the attack depends on **what we overwrite**.

We started with an ordinary integer and progressed to a code pointer, allowing us to hijack control flow.

Along the way, we introduced several concepts that will appear repeatedly throughout the rest of the course:

* buffers and buffer overflows,
* virtual addresses,
* the stack,
* raw memory and byte representations,
* little-endian ordering,
* x86-64 calling conventions,
* function pointers and code pointers,
* the instruction pointer,
* control-flow hijacking,
* PIE and ASLR,
* GDB/GEF,
* and pwntools.

Most importantly, start developing the habit of treating exploit development as a process of **building and verifying a precise model of the program**.

When you get stuck, do not guess.

Draw the memory.

Look at the bytes.

Step through the instructions.

Check your assumptions.
