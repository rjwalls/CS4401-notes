---
title: "Lecture Notes: Basics of Buffer Overflows"
date: 2020-01-01 02:00:00
categories: notes lecture
layout: post
challenges: stack0r-64 stack1r-64 stack2r-64 stack0r-guessdown stack2r-warp-drive
---

Over the first few lectures, we are going to jump straight into binary exploitation. We will start with a very simple bug---a stack-based buffer overflow caused by `gets()`---and, somewhat intentionally, keep exploiting essentially that same bug for several challenges.

Why keep using the same bug? Because the buffer overflow itself is not the hard part. I want to keep that part simple while we introduce the other concepts that you need in order to exploit binaries:

* How is memory organized?
* What exactly is a buffer overflow?
* How do we figure out where objects actually live in memory?
* What is endianness, and why does it matter?
* What is a code pointer?
* What does it mean to hijack a program's control flow?
* How do we use GDB, GEF, and pwntools to answer these questions instead of guessing?

The targets will gradually become more interesting. Roughly speaking, we will progress from

1. changing an integer to *anything other than zero*,
2. changing an integer to an *exact value*, and
3. changing a **function pointer**, which lets us begin manipulating the program's control flow.

Later, we will take another step and target a particularly important code pointer: the **saved return address**. But we have a fair amount of groundwork to cover before we get there.

## Getting Started with `stack0`

Consider the basic structure of our first challenge:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
    volatile int unsecured = 0;
    char buffer[{{buffsize}}];
    FILE *fp;

    gets(buffer);
    fp = fopen("./flag.txt", "r");

    if (unsecured != 0) {
        printf("The 'unsecured' variable has been changed!\n");
        printf("Warning! Multiool is no longer in security mode\n");
        fgets(buffer, 64, fp);
        printf("flag: %s\n", buffer);
    } else {
        printf("Try again?\n");
    }
}
```

Our goal is to convince this program to print the contents of `flag.txt`.

There is an obvious problem: `unsecured` starts at zero, the program never assigns another value to it, and the flag is only printed if `unsecured != 0`.

Under the program's intended execution, we lose.

Fortunately, this is a binary exploitation course. We are not limiting ourselves to the program's intended execution.

### What does it mean to exploit a binary?

For our purposes, **exploiting a binary** means taking advantage of some flaw in a program to make the program behave in a way that its programmer did not intend.

For `stack0`, our intended behavior is particularly simple:

```c
unsecured = 0;
...
if (unsecured != 0) {
    ...
}
```

The programmer clearly expects `unsecured` to remain zero. Our job is to make that assumption false.

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
gets(buffer);
```

`gets()` reads from **standard input**, or `stdin`. If you execute the program from a terminal and type some characters, those characters are being provided through standard input.

Later challenges will force us to think about other input mechanisms. For example, environment variables matter in `stack1`.

For now, though, `gets()` gives us exactly the interface we need.

## The Bug: `gets()`

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

For `stack0`, the important one is the stack because both of these are local variables:

```c
volatile int unsecured = 0;
char buffer[...];
```

Local variables normally live in the function's **stack frame**.

We will give stack frames a much more careful treatment when we get to return addresses. For now, we mostly care that `buffer` and `unsecured` are both sitting somewhere in stack memory.

## Exploiting `stack0`

We now have all the pieces we need.

`gets()` starts writing at the beginning of `buffer` and continues writing toward higher addresses.

If `unsecured` happens to be at a higher address than `buffer`, then sufficiently long input can run off the end of `buffer` and eventually begin overwriting `unsecured`.

Conceptually, perhaps memory looks like this:

```text
◄──────────────── Lower addresses       Higher addresses ────────────────►

+-------------------------------+----------------+
|            buffer             |   unsecured    |
+-------------------------------+----------------+
^
gets() starts writing here ──────────────────────►
```

If we keep writing long enough:

```text
+-------------------------------+----------------+
| A A A A A A A A A A A A A A | A A A A        |
+-------------------------------+----------------+
                                  ^
                                  unsecured is no longer zero
```

And suddenly:

```c
if (unsecured != 0)
```

is true.

That is our first exploit.

The attack itself is simple:

> **Use a buffer overflow to change some adjacent piece of memory to a value that is useful to us.**

That basic description will continue to apply to the next few challenges. We will just keep choosing more interesting things to overwrite.

## Don't Assume the C Source Tells You the Memory Layout

There is an important catch.

Consider:

```c
int unsecured;
char buffer[64];
```

You should **not** conclude from the order of the C declarations that the compiler must put `unsecured` immediately before or after `buffer`.

The compiler determines the actual layout.

It could look like this:

```text
+----------------------+------------+
|       buffer         | unsecured  |
+----------------------+------------+
```

or:

```text
+------------+----------------------+
| unsecured  |       buffer         |
+------------+----------------------+
```

or:

```text
+----------------------+-----+----------------+------------+
|       buffer         | ??? |      ???       | unsecured  |
+----------------------+-----+----------------+------------+
```

There may be padding. There may be other saved values. Compiler choices may change between builds.

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

elf = ELF('./stack0-64')
context.binary = elf
context.log_level = 'debug'

p = gdb.debug('./stack0-64', gdbscript='''
    break stack0.c:12
    continue
''')

p.sendline(b'a' * 133)

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

Finally:

```python
p = gdb.debug(...)
```

runs the program under GDB/GEF so that we can inspect the process while our *actual exploit script* interacts with it.

That is generally much more useful than having one test input in GDB, a different Python one-liner outside GDB, and a third version embedded in an exploit script.

### Looking at memory

Once we have stopped the program after `gets()`, commands like these become useful:

```gdb
info locals
```

```gdb
p &buffer
```

```gdb
x/s &buffer
```

```gdb
x/160bx &buffer
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

For example, imagine we see something like this near the call to `gets()`:

```asm
lea    rax,[rbp-0x90]
mov    rdi,rax
call   gets@plt
```

The x86-64 System V calling convention specifies that the first function argument is placed in `rdi`.

`gets()` takes one argument: the address of the destination buffer.

Therefore, immediately before the call:

```text
RDI = address of buffer
```

That is incredibly useful.

Rather than trying to reverse-engineer every instruction in `main`, we can often go straight to the vulnerable function call, place a breakpoint nearby, and examine the argument register:

```gdb
p/x $rdi
```

or:

```gdb
x/64bx $rdi
```

This is a recurring exploit-development strategy:

> **Use what you already know about the function interface and calling convention to find the information you care about.**

You do not have to understand every instruction in a binary before you can exploit it.

You do, however, need to understand the instructions that matter to your attack.

## `stack1`: Exact Values and Endianness

In `stack0`, life is easy.

We do not particularly care what value we place in `unsecured`. We only need:

```c
unsecured != 0
```

So one `a` byte is perfectly adequate:

```text
0x61
```

The next challenge makes things more interesting.

Instead of changing an integer to *anything other than zero*, suppose we need to make an integer equal some exact value:

```c
if (modified == 0x0d0a0d0a) {
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

If the vulnerable program reads from an environment variable, pwntools can set that environment for us:

```python
from pwn import *

p = process(
    './stack1-64',
    env={
        "DEBUG": b"a" * 108 + p32(0x0d0a0d0a)
    }
)

p.interactive()
```

Again, the high-level question is:

> What data in this process can I control, and where does the program copy that data?

That question generalizes much better than memorizing a particular exploit string.

## `stack2`: From Data Corruption to Control-Flow Hijacking

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
win()
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
normal_function
```

but we corrupt a function pointer so that it instead points to:

```text
win
```

Then the indirect call eventually causes:

```text
RIP = address of win
```

We have changed the program's control flow.

This is a significant step up from changing an integer.

And it is the basic idea behind many of the attacks we will study for the rest of the course.

## Finding the Target Address

To overwrite a function pointer with the address of `win`, we first need to know the address of `win`.

If the address is fixed, GDB may make this very easy:

```gdb
p &win
```

or:

```gdb
info address win
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

You might see output resembling:

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
win = 0x7aa
```

That does **not** necessarily mean that `win()` is really executing at virtual address `0x7aa`.

Once the process is running, you might instead find something like:

```text
0x5555555547aa
```

The useful observation is that the offset inside the executable remains the same:

```text
base + 0x7aa
```

That is a consequence of the coarse-grained randomization we will discuss in our ASLR notes.

For an early challenge, we may simply provide you with an **information leak** that reveals an address from the current process. Once we have one known address in a region, fixed offsets often let us calculate the address we actually want.

The important point for now is not to memorize an ASLR bypass.

It is to start asking:

> **Where did this address come from, and is it still valid for the process I am exploiting?**

## Developing the Exploit Systematically

By this point, a basic exploit might have a structure like:

```python
from pwn import *

elf = ELF('./stack2-64')
context.binary = elf
context.log_level = 'debug'

p = gdb.debug('./stack2-64', gdbscript='''
    break main
    continue
''')

offset = ...
target = ...

payload = flat({
    offset: target
})

p.sendline(payload)
p.interactive()
```

The `...` values are not numbers that we should fill in through inspired guessing.

They are questions that we answer.

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
/problems/stack2r-64/
```

You can do:

```bash
cd /problems/stack2r-64/
python3 ~/exploits/stack2.py
```

If the script opens:

```python
ELF('./stack2r-64')
```

then `./stack2r-64` is resolved relative to the **current working directory**, not relative to the directory containing your Python script.

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

* `stack0r-64`: first stack-based buffer overflow and adjacent data corruption.
* `stack1r-64`: the same basic bug, but now the overwritten value must be exact and the attacker-controlled input comes from an environment variable.
* `stack2r-64`: the same basic bug, but the corrupted object is a function pointer, which introduces code pointers and control-flow hijacking.

It also sets up:

* `stack0r-guessdown`: the same early questions in a less comfortable setting: what input do I control, what object can I corrupt, and how do I measure the offset?
* `stack2r-warp-drive`: the same function-pointer model, plus the practical question of where a useful target address comes from in the process I am exploiting.

The next set of notes on stack frames and return addresses will pick up with `stack3r-64`.

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
