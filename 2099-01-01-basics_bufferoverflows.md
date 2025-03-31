---
title: "Lecture Notes: Basics of Buffer Overflows"
date: 2020-01-01 02:00:00
categories: notes lecture
layout: post
challenges: stack0 stack1 stack2 stack3
---

Over these first few lectures, we will jump straight into binary exploitation, starting with the `stack0` challenge. We’ll begin with the basics of stack-based buffer overflows and gradually work our way up to more complex examples, like the classic stack-smashing attack. Along the way, we'll answer some key questions, such as:

- What does it mean to exploit a binary?
- How are objects laid out in memory?
- How can we use basic buffer overflows to manipulate memory?
- What is a return address?
- How can we use disassembly to figure out the layout of variables on the stack?
- What are `setuid` binaries?
- What is privilege escalation?

### Getting Started with `stack0`

Let’s start with the `stack0` challenge. Below, you’ll find the source code for this challenge binary. Our goal is to figure out how to exploit it. But what does "exploiting" a binary really mean? Simply put, it’s about finding and using vulnerabilities in the code to make the program do something it wasn’t intended to do.

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
  fp  = fopen("./flag.txt", "r");

  if(unsecured != 0) {
      printf("The 'unsecured' variable has been changed!\n");
      printf("Warning! Multiool is no longer in security mode\n");
      fgets(buffer, 64, (FILE*)fp);
      printf("flag: %s\n", buffer );
  } else {
      printf("Try again?\n");
  }
}
```

For this challenge, our goal is to manipulate the program so it executes the code that prints out the contents of `flag.txt`. "Ah!" you say, "the program only prints out the line of text if the value of `unsecured` is not zero, but the program sets `unsecured` to zero and never changes it, so it’s impossible to print `flag.txt` under normal execution." You’re absolutely right, my clever student. But we’re not going to rely on normal execution---we're going to think outside the box.

As an aside, you might wonder, "Why can’t we just print out the contents of `flag.txt` ourselves?" That’s an excellent question. I want you to try it. Connect to the course server via `ssh`, navigate to the challenge directory using `cd /path/to/challenge`, and run `cat flag.txt`. What happened? You probably saw an error message telling you that you don’t have permission to read the flag file. While we don’t have permission to read the flag file directly, the challenge binary does. This is thanks to a security mechanism called **setuid**, which we’ll discuss later in this lecture.

Another interesting detail is the `volatile` keyword. Compilers are smart and they optimize programs to run faster. Without `volatile`, the compiler might notice that `unsecured` never changes and decide to optimize it out completely, making the variable disappear. The purpose of the `volatile` keyword is to prevent the compiler from making this optimization, ensuring that `unsecured` is always stored in memory and can be changed during program execution.

#### Our first exploit

Broadly speaking, our recipe for success requires two key ingredients. First, the program must have a bug. Second, we need a way to provide input to the program that triggers that bug. Let’s start by figuring out how we can deliver our malicious input.

#### Ingredient: Supplying malicious input

There are many ways to provide input to a program. In the `stack0` challenge, the most straightforward way is through the `gets()` function call.

The `gets()` function is a C standard library function that reads a line of text from the standard input (often referred to as `stdin`). `stdin` is the default input stream for a program, usually connected to the keyboard in a terminal. When you run a program and type something, `stdin` is where that input goes. The `gets()` function takes this input and stores it in a buffer (a specific area in memory), stopping when it encounters a newline character or the end of the input.

However, even for this simple challenge binary, there are other ways we can provide input. For example, we control the command-line arguments passed to the program, like this: `./stack0 "Arg1" "Arg2" ...`. Although this program doesn’t use command-line arguments, they are still passed to the program when it runs.

Command-line arguments are the values you provide after the program's name when you run it from the terminal. These arguments are passed to the program by the operating system and can be used by the program to perform different tasks based on the input provided. In `stack0`, these arguments are ignored, but they will prove useful for other challenge binaries.

In addition to command-line arguments, we can also supply input via environment variables.

Environment variables are key-value pairs in the operating system that affect the behavior of running processes. They provide a way to pass configuration information to programs. For example, environment variables can tell a program where to find specific files or set preferences for how it should run. While we don't need to use environment variables for `stack0`, we will use them for other challenges in this course.

In other programs, input might come from files on disk, network connections, external sensors, and more. But for `stack0`, using `stdin` is our best bet because that’s where the `gets()` function is looking for input.

#### Ingredient: The bug

Before we dive into the specifics, let’s first define what we mean by a "bug." A _bug_ is an error or flaw in a program that causes it to behave in unintended ways. When this bug can be exploited by an attacker to cause harm or gain unauthorized access, it’s considered a _vulnerability_. In this course, we’ll often use the terms "bug" and "vulnerability" interchangeably because, from a security perspective, we’re particularly interested in those bugs that can lead to vulnerabilities.

Finding bugs often requires time, patience, and a bit of intuition—the kind of intuition that comes with experience. One of the goals of this course is to help you start developing that intuition, so by the end, you’ll be better equipped to identify potential vulnerabilities in code.

A good place to start is by looking for some common mistakes that programmers make. Some of these mistakes include improper handling of input, mismanaging memory (like with arrays or pointers), and neglecting to validate data. These errors can often lead to vulnerabilities like buffer overflows, which is what we’re going to explore next.

Since programmers often make mistakes when working with arrays, let’s take a closer look at how the `buffer` is used in this program. We’ve mentioned the `gets()` function before, and it’s important here because it’s both a way to take input (our first ingredient) and a mechanism that can lead to bugs when working with arrays. This makes it a great place to start our investigation.

To understand if there’s a bug we can exploit, we need to understand how `gets()` works. This brings us to an important theme of this course: to exploit systems effectively, we must deeply understand how they work. That means knowing how functions like `gets()` operate under the hood.

To learn more about `gets()`, we can consult the _man page_ (short for "manual page"), which provides documentation for command-line utilities and functions. A man page typically includes details on how the function works, its parameters, return values, and any known issues or bugs. You can access a man page by typing `man gets` in your terminal—a command-line interface where you can type commands directly to the operating system. It's a good idea to read the man page on a system similar to your target, as different operating systems may have slightly different versions of standard library functions.

From the man page, we learn that `gets()` reads a line from `stdin` (standard input) into a buffer until it encounters a newline character or EOF (end-of-file). A newline is an ASCII character with a specific byte value that indicates the end of a line of text, while EOF marks the end of the input data.

Man pages contain a wealth of information, and sometimes they include a `BUGS` section, which is particularly useful when looking for potential vulnerabilities. In the case of `gets()`, the `BUGS` section warns:

> Never use gets(). Because it is impossible to tell...how many characters gets() will read.

This warning might sound alarming if you’re a programmer, but if you’re an attacker, it’s promising! But what does this warning really mean? To understand the implications, we need to revisit how memory is organized in a program.

You might have already covered memory layout in an Operating Systems course, but it’s worth reviewing because understanding how programs use memory—including how variables and buffers are arranged in memory—is crucial for identifying vulnerabilities.

An array is essentially a sequence of contiguous bytes in memory. More generally, a **buffer** is any contiguous region of memory associated with a variable. For example, an `int` variable typically uses a buffer of 4 bytes, while a `char` variable uses a buffer of 1 byte. The size of the buffer for an array depends on both the data type of the array and the number of elements it contains.

Some texts define buffers more narrowly, considering only variable-length data as buffers. For example, they might not refer to an `int` as a buffer, but in this course, we’ll use the broader definition.

Using the information from the man page, we can conclude that in the `stack0` example, `gets()` will copy input from `stdin` into the `buffer` character array (the buffer named `buffer`). For instance, if we run `stack0` and supply `aa` as input, `gets()` will store three bytes in `buffer`: `a`, `a`, `\0` (the null terminator). But what happens if we provide more input than the size of the `buffer`? For example, what if we supply a string of 100 characters?

Let’s put on our thinking caps and consider what might happen. One possibility is that `gets()` might realize the input is too large for the buffer and simply refuse to read anything. Another possibility is that `gets()` could stop reading once the buffer is full, only writing as much data as the buffer can safely hold.

However, for either of these possibilities to work, `gets()` would need to know both the size of the buffer and the size of the input data. But, as the warning in the man page points out, `gets()` doesn’t have this information!

The actual behavior of `gets()` is the third possibility: it will happily keep writing as much data as it’s given, even if that means going past the end of the array and overwriting whatever memory happens to be next. Further, this is an easy mistake for a programmer to make. This is what makes `gets()` so dangerous and why it was a common source of vulnerabilities.

This type of bug is called a **buffer overflow**. Buffer overflows have been widely exploited, with some famous examples including the Morris Worm in 1988 and the Code Red Worm in 2001, both of which exploited buffer overflows to spread. These vulnerabilities were common in early software development and remain a significant concern today. Buffer overflows fall into a broader category of issues known as memory errors, which we’ll explore throughout this course.

One of the fascinating (and frustrating) aspects of memory errors is that they don’t always cause immediate problems. The program might continue running without any apparent issues, only to crash later when the corrupted memory is used. For example, `stack0` will keep executing for a while after the `gets()` call before showing any signs of trouble. If you’ve ever had a program crash seemingly out of nowhere, it might have been due to a memory error like this, and you might already understand how difficult these errors can be to debug.

To truly grasp the dangers of buffer overflows, we need to understand what’s _next to_ `buffer` in memory. And that will require us first to review the concept of a virtual address space.

### Virtual Address Space of a Process

First, let’s define what a _process_ is. A process is an instance of a running program. It includes the program’s code, its current activity (like the values of program counters and registers), and the resources allocated to it by the operating system, such as memory and file handles. Essentially, when you run a program, the operating system creates a process to manage that program’s execution.

The operating system, with some help from the computer’s hardware, provides processes with many useful abstractions. One of the most important is the **virtual address space**. This abstraction makes memory appear to each process as a contiguous array of bytes, allowing the process to use memory as if it had the entire memory space to itself.

The operating system also provides other abstractions, like files, which allow processes to read and write data without needing to know how that data is physically stored on a disk. Similarly, a process itself can be seen as an abstraction of the CPU (Central Processing Unit), giving the program the illusion that it has the CPU all to itself, even though multiple processes are sharing the CPU.

#### Basics of Virtual Memory

In virtual memory, each location in memory can store a byte, and each byte is associated with a unique _address_---a number that identifies its location. This address is what the program uses to access the memory.

Each **process** running on a machine has its own virtual address space. This makes physical memory (RAM) much easier to use because a process doesn’t need to know where its data is actually stored in the physical RAM. Instead, the operating system (with lots of hardware help) handles the details, making memory management simple for the process.

When we talk about "memory" in the context of an executing process, we’re usually referring to this virtual memory—the abstracted view provided by the operating system.

Virtual memory is also the foundation of **process isolation**—a concept where each process is kept separate from all other processes on the system. This isolation is crucial for security and stability because it prevents processes from interfering with each other. For example, if one process crashes, it won’t affect the others because they are isolated in their own virtual address spaces. The operating system (again, with hardware help) ensures that each process is safely contained within its own memory boundaries.

#### Memory Regions in a Process

A process typically divides its virtual memory into different regions, each serving a specific purpose. Some of the most important regions include the stack, the heap, and the text sections.

- The **stack** is a region of memory used by each thread (a thread is a smaller unit of a process that can run independently) to keep track of the thread’s execution state. This includes local variables, information about which functions have been called, and other temporary data. The stack for the main thread (the primary thread that begins executing when the program starts) also includes the command-line arguments and environment variables.
- The **heap** is a region used for dynamically allocated memory. For example, when you use `malloc()` to allocate memory in a C program, that memory comes from the heap. Managing this memory correctly is challenging for programmers and is another common source of errors, which we’ll discuss in future lectures.
- The **text** section of memory stores the compiled program code. This area is often marked as read-only to prevent the code from being accidentally (or maliciously) modified during execution.

The layout and size of the address space can vary depending on both the operating system and the hardware. For instance, a 32-bit process has a 32-bit virtual address space, where each address is 4 bytes long (or 8 hex digits). A 64-bit process, on the other hand, has a 64-bit virtual address space, with addresses that are 8 bytes long.

If we visualize the layout of a 32-bit virtual address space on a Linux system, it might look something like this:

```
0xff ff ff ff
cmd env (set at process start, lives on main thread's stack)
stack (grows toward lower adddresses, bottom is fixed)
heap (managed by malloc)
text (code, read only)
0x00 00 00 00
```

Here, memory is represented with the address `0x0` at the bottom and the highest address at the top. We could visualize this layout in different orientations, but the concept remains the same. It’s also common to use hexadecimal (hex) notation to represent memory addresses. Hex notation is a base-16 numbering system that uses the digits 0-9 and the letters A-F. It’s often used in computing because it’s more compact and aligns well with the binary system.

In this course, we’ll work with both 32-bit and 64-bit binaries. As mentioned, the former uses a 32-bit virtual address space, and the latter uses a 64-bit virtual address space. A quick way to determine whether a binary targets a 32-bit or 64-bit architecture is to use the `file` command: `file stack0`.

#### Sparse Virtual Address Space

Another important concept to introduce here is the idea of a sparse virtual address space. In practice, not all addresses in a virtual address space are used. The operating system only allocates physical memory to parts of the virtual address space that a process is actively using. This helps conserve memory and allows processes to have larger address spaces than the physical memory available on the system.

### Exploiting the Buffer Overflow in `stack0`

After reviewing the behavior of `gets`, and the concepts of buffers and virtual memory, we’re ready to craft our malicious input. This type of input is often called an _exploit string_ because it’s often delivered as a string of characters; it may also be referred to as a _payload_.

As we discussed earlier, `gets()` will cheerfully copy whatever input we provide to `stack0` (to the extent that machines are capable of feeling joy), even if that input is larger than the buffer, which allows us to overwrite the memory next to `buffer`. So, what’s adjacent to `buffer`? In this challenge binary, `buffer` is a local variable, and local variables are stored on the stack. Remember, the stack is a region of memory that keeps track of a thread’s execution state, including local variables.

Fortunately, the `unsecured` variable---the one we need to change to solve the challenge---is also a local variable. This means it’s stored on the stack and could be overwritten if we overflow the `buffer`. Let’s test this by overflowing the `buffer` to see what happens.

```bash

python3 -c "print('a' * 100)" | ./stack0

```

The command above generates a string of 100 `a` characters using Python and pipes this string into the `stack0` program. The `| ./stack0` part runs the `stack0` program and feeds the generated string as input.

Running this command gives us the response we were hoping for: "you have changed the 'unsecured' variable." This confirms that `unsecured` was indeed adjacent to `buffer` in memory and was overwritten by our large input string. We’ve successfully crafted an exploit string!

However, there are some important caveats to keep in mind. These mean that exploiting other challenge binaries won’t necessarily be as straightforward.

#### Important Caveats

The first caveat is that we got a bit lucky with the placement of the `unsecured` variable. `gets()` writes to `buffer` starting at the beginning and continues writing to higher memory addresses one character at a time. The fact that `unsecured` was located at a higher address than `buffer` allowed our overflow to reach and overwrite it. However, the compiler decides the layout of variables on the stack, so it could have easily placed `unsecured` before `buffer`, making this particular exploit ineffective.

To summarize visually:

To summarize visually, here’s what we assumed the memory layout looked like:

```
◄──────────────  Lower Addrs              Higher Addrs ──────────────────►

┌─────────────────────────────────────────────────────────────────────────┐
│           |                 buffer                        | unsecured   │
└─────────────────────────────────────────────────────────────────────────┘
```

But the compiler could have arranged the variables differently, like this:

```
◄──────────────  Lower Addrs              Higher Addrs ──────────────────►

┌─────────────────────────────────────────────────────────────────────────┐
│  unsecured |                 buffer                        |            │
└─────────────────────────────────────────────────────────────────────────┘
```

Or even something more complicated:

```

◄──────────────  Lower Addrs              Higher Addrs ──────────────────►

┌─────────────────────────────────────────────────────────────────────────┐
│          buffer                 | ???     |   ?????       | unsecured   │
└─────────────────────────────────────────────────────────────────────────┘
```

The second caveat is one we mentioned earlier: compilers are clever and always looking for ways to optimize performance. Because there’s no way to update the `unsecured` variable in the normal flow of the program, the compiler might decide to optimize it out entirely, replacing it with hard-coded zeroes instead. To prevent this, `stack0` includes the `volatile` keyword, which forces the compiler to keep the `unsecured` variable in memory, ensuring it can be changed by our exploit.

This brings us to an important point: **if we want to know what code will actually execute, we need to look at the disassembly of the binary.** The compiler transforms the C code into machine code, and to truly understand what the program is doing, we need to examine the binary directly.

### Looking at the Disassembly

Let’s dive deeper into the `stack0` binary by using `gdb`, a powerful debugging tool that allows us to inspect the instructions and memory of a running program.

In this section, we’ll cover several key points:

- What `gdb` is and how it’s used for debugging and disassembling binaries.
- The difference between Intel and AT&T assembly syntax.
- How to identify the locations of key variables like `unsecured` and `buffer` in memory.
- How to use assembly instructions and registers to understand program behavior.

#### Introduction to `gdb` and Disassembly

`gdb` (GNU Debugger) is a tool used to analyze and debug programs. It allows you to set breakpoints, examine memory, and disassemble code, which means converting the binary machine code back into a human-readable assembly language format. This process is called _disassembly_ and is crucial for understanding what the compiled code is actually doing, especially when you don't have access to the original source code.

It’s important to note that some binaries come with debug symbols---extra information that makes debugging easier by mapping the binary back to the original source code. However, our example doesn't include debug symbols, so we’ll need to rely solely on the disassembled output.

#### Switching to Intel Assembly Syntax

Before we start, let’s change how `gdb` displays assembly code. Specifically, we’ll switch from the default AT&T syntax to Intel syntax, which is just a different way of writing the same assembly instructions. Both syntaxes are common, but we’ll use Intel syntax here because it’s more widely used in exploit documentation and easier to read for many people.

The first thing we are going to do is change how `gdb` displays assembly.
Specifically, we are telling `gdb` to use Intel syntax rather than AT&T syntax. We do this purely based on personal preference. TODO: Define what intel and at&t syntax are and why both are common.

You can switch to Intel syntax in `gdb` with the following command:

```
set disassembly-flavor intel
```

Now that we’re in Intel syntax mode, it’s a good idea to brush up on the basics of x86 assembly. Here’s a quick overview:

- An **instruction** in assembly is a command that tells the CPU what to do, like moving data or performing arithmetic.
- An **assembly mnemonic** is the shorthand name for an instruction, like `mov` for "move" or `add` for "add."
- **Operands** are the values or locations that the instruction operates on. For example, in `mov eax, ebx`, `eax` and `ebx` are operands.
- A **register** is a small, fast storage location inside the CPU, like `eax` or `esp`, used to hold data temporarily during computation.

In Intel syntax, the format is typically `instruction destination, source`. For example, the instruction `mov DWORD PTR [esp+0x5c], 0x0` means "move the 32-bit value `0x0` into the memory address at `esp+0x5c`." Here, `esp` is the stack pointer register in 32-bit x86, pointing to the top of the stack, and `0x5c` is the offset in hexadecimal added to that address.

Another common instruction is `lea`, which stands for "load effective address." Instead of loading the data at a given address, `lea` loads the address itself into a register.

Here are some of the most important registers you’ll encounter:

- **EIP (Extended Instruction Pointer):** This register points to the next instruction that the CPU will execute. It’s crucial for controlling the flow of the program. Every time an instruction is executed, the EIP is updated to point to the next one, ensuring the program runs sequentially unless a jump or call instruction changes the flow.

- **EBP (Extended Base Pointer):** The EBP register is typically used as a reference point for the current stack frame, which contains the local variables and function arguments for a particular function. It helps the program keep track of the stack’s layout, especially when functions call other functions. When a function starts, EBP is set to the current value of ESP (the stack pointer), and it remains unchanged throughout the function, making it easier to access variables relative to the stack frame.

- **ESP (Extended Stack Pointer):** The ESP register points to the top of the stack, which is a region of memory used for storing temporary data, such as function parameters, return addresses, and local variables. The stack grows and shrinks as functions are called and return, with ESP always pointing to the current top of the stack.

#### Disassembling the `main` Function

Now, let’s take a look at the `main` function in `stack0`:

```
(gdb) disassemble *main
0x75a <+0>:     push   rbp
0x75b <+1>:     mov    rbp,rsp
0x75e <+4>:     add    rsp,0xffffffffffffff80 // Allocating 128 bytes on stack
0x762 <+8>:     mov    DWORD PTR [rbp-0x74],edi
0x765 <+11>:    mov    QWORD PTR [rbp-0x80],rsi
0x769 <+15>:    mov    DWORD PTR [rbp-0xc],0x0 // Initialize 'unsecured' flag to 0
0x770 <+22>:    lea    rax,[rbp-0x70]         // Load address of buffer
0x774 <+26>:    mov    rdi,rax                // Set up parameter for gets()
0x777 <+29>:    mov    eax,0x0
0x77c <+34>:    call   0x620 <gets@plt>       // Vulnerable call to gets() - no bounds checking
0x781 <+39>:    lea    rsi,[rip+0x100]
0x788 <+46>:    lea    rdi,[rip+0xfb]
0x78f <+53>:    call   0x630 <fopen@plt>
0x794 <+58>:    mov    QWORD PTR [rbp-0x8],rax
0x798 <+62>:    mov    eax,DWORD PTR [rbp-0xc] // Load 'unsecured' value
0x79b <+65>:    test   eax,eax                // Check if 'unsecured' is 0
0x79d <+67>:    je     0x7e6 <main+140>       // Jump if 'unsecured' is still 0
```

In this output:

- The first column shows the memory location of each instruction in the `main` function. These addresses are in the text section of the virtual address space, where the program code resides.
- The second column shows the byte offset of each instruction from the start of `main`. For example, `<main+64>` means that the instruction is 64 bytes (in decimal) from the start of `main`. Yeah, you'll see offsets in both hexadecimal and decimal, so it is best to get used to both now.
- The remaining columns show the assembly mnemonic and operands for each instruction.
- I've also added some additional comments to the code that won't be there in the actual output.

It’s important to note that the assembly mnemonics shown here are not what’s stored in memory. Instead, the mnemonics are human-readable versions of the binary machine code, made visible by using a disassembler like `gdb`.

Our goal is still to determine where `unsecured` and `buffer` are relative to each other in memory, but now we need to look at their positions relative to the base pointer register, `rbp`.

We can identify the memory location of the `unsecured` variable by finding the instructions that correspond to the C code `unsecured = 0;` and `if(unsecured != 0)`. In this disassembly, the `mov DWORD PTR [rbp-0xc],0x0` instruction at address `0x769` initializes `unsecured`, and the sequence of `mov`, `test`, and `je` instructions starting at `0x798` checks its value. This tells us that `unsecured` is located at `rbp - 0xc`.

Similarly, we can find the location of `buffer` by looking for the call to `gets()`. In the disassembly, the instruction at `0x770` loads the address `rbp-0x70` into `rax`, which is then passed to `gets()` via the `rdi` register (following the x86-64 **calling convention**). This tells us that `buffer` is located at `rbp - 0x70`.

Keep in mind that compiler optimizations can sometimes result in disassembly that looks very different from the original source code. The compiler might rearrange or even eliminate certain instructions to improve performance.

Now that we know `unsecured` is at `rbp-0xc` and `buffer` is at `rbp-0x70`, we can calculate the relative positions. A quick calculation (`0x70 - 0xc`) shows that the start of `buffer` and the start of `unsecured` are 100 (0x64) bytes apart, instead of 64 bytes in the original version.

To exploit this, we'd need to modify our overflow string:

```bash
python3 -c "print('a'*100);" | stack0
```

You might wonder if this code produces a string long enough to overwrite `unsecured` given that it only writes 64 'a' characters. It does! This input will overwrite the first byte of `unsecured` because the Python code `print('a'*64)` produces a 65-character string---64 `a` characters followed by a newline character (`0x0a`), which is automatically added by `print`. The newline character is what overwrites `unsecured`. Let’s use `gdb` to see the impact of this input on a running process:

```bash
$ python -c "print('a'*100)" > input.txt

$ gdb stack0
disass main // view the disassembly for the main function
break *0x79e // put a breakpoint right after the check of unsecured
r < input.txt // run the binary with the long input
x/s $rbp-0x70 // to see the input string in stack memory
x/104bx $rbp-0x70 // to see input string bytes in hex
```

In this output, we see a series of commands that we used to interact with the program during debugging. Let’s break down what each of these commands does:

- **Breakpoint (`break`)**: A breakpoint is a marker you set in your code that tells the debugger to pause the program’s execution when it reaches a specific point. This allows you to inspect the program's state at that exact moment, making it easier to diagnose issues or understand what the code is doing. In this case, we set a breakpoint at a particular address in the `main` function.

- **Run (`r`)**: After setting a breakpoint, we use the `r` (short for "run") command to start the program. We pass the file "input.txt" as input, which is directed to the program’s standard input (stdin). The program reads this input as if it were typed directly into the terminal.

- **Examine (`x`)**: The `x` command in `gdb` is used to examine the contents of memory at a specific address. We can specify how we want the memory to be interpreted:

  - **`x/s`**: This option interprets the memory content as a string and shows the characters stored at that memory address.
  - **`x/68bx`**: This option interprets the memory content as 68 consecutive hexadecimal byte values. It’s useful for seeing the exact byte-level representation of the data stored in memory.

After running these commands, you should see the value `0x0a000000` right after the sequence of `'aaaa...'` in memory. This value represents the `unsecured` variable that we altered by writing a newline character (`0x0a`).

It’s important to note that in this context, we’re working with integers (`int`), which are four bytes long. The memory representation we’re seeing is in _little-endian_ format. In little-endian systems, like those using the Intel x86 architecture, the least significant byte (the "smallest" part of the number) is stored first, at the lowest memory address. So, when we see `0x0a000000`, it means that the byte `0x0a` is stored first, followed by three `0x00` bytes.

Understanding and accounting for endianness is a major challenge for many students at the start of this course. You’ll encounter this concept repeatedly as we continue, so it’s important to get comfortable with how data is stored and represented in memory on little-endian systems.

#### Summary

In this section, we’ve explored the assembly-level details of the `stack0` program. We’ve learned how to identify the locations of important variables like `buffer` and `unsecured` in memory, and how these positions relate to the stack pointer. Additionally, we’ve discussed how we can use tools like `gdb` to disassemble code and inspect memory.
Understanding these concepts is crucial for analyzing and crafting exploits, as it allows us to comprehend and manipulate how a program behaves at the lowest level.

Now that we’ve got a solid grasp of the program at the assembly level, let’s consider some other useful information that’s stored on the stack.

### Introducing Return Addresses

When exploiting the buffer overflow in `stack0`, you might encounter a `segmentation fault` error. This happens when our input overwrites not just the `unsecured` variable but also other adjacent values in memory. One of these critical values is the **return address**. When the return address is corrupted, it triggers the segmentation fault.

To understand the return address, let’s tie it back to what we’ve discussed about memory and code execution. Recall that the code executed by the CPU is a sequence of machine instructions, which we often visualize as assembly language. The CPU uses a special register called the **instruction pointer** (IP) to keep track of the next instruction to execute. When a function is called, the instruction pointer jumps to the first instruction of that function.

However, because functions need to return to the point they were called from, the current value of the instruction pointer (the return address) must be saved before the jump. This saved return address is stored in memory---specifically, on the stack---so that the CPU can resume execution at the correct point after the function completes.

Now, we can see what causes the segmentation fault. When we overflow the buffer, we overwrite `unsecured` and eventually the return address. When the function tries to return, it loads the corrupted address (now a series of `aaaa` characters, or `0x61616161` in hexadecimal) into the instruction pointer. Since this address doesn’t point to valid code, the hardware flags an error, notifies the operating system, and the process is terminated.

We can avoid this segmentation fault in `stack0` by adjusting our exploit to only overwrite the `unsecured` variable, leaving the return address intact. But a more intriguing question is: What can we do if we can _control_ the return address? By controlling the return address, we can redirect the program's control flow---essentially, we can tell the program to execute code of our choosing. If done skillfully (and with a bit of luck), this can lead to _arbitrary code execution_.

Arbitrary code execution means that an attacker can run any code they choose on a target machine. Imagine the possibilities: with this kind of control, an attacker could steal sensitive information, install malicious software, or even take full control of the system. It’s easy to see how dangerous this could be for the machine's owner.

One of the main goals of this course is to help you understand how attackers exploit binaries to achieve arbitrary code execution. By grasping these attacks and the vulnerabilities that enable them, you'll be better equipped to design and implement effective defenses.

### Understanding Binaries

Now is a good time to look at some of the details of binaries---what they are, how they work, and some complications. A _binary_ is the compiled form of a program that the operating system (OS) loads and runs as a process. They are also referred to as _executable files_. In this section, we’ll cover several key topics:

- What ELF (Executable and Linkable Format) binaries are and how they’re loaded on Linux.
- How file permissions and the `setuid` bit impact security.
- The concept of Position Independent Executables (PIE) and how they affect exploit development.

#### ELF (Executable and Linkable Format)

**ELF (Executable and Linkable Format)** is a standard format for executable files on Linux. When you run a program on Linux, the OS takes several steps to load the ELF binary into memory and start the process. Let’s break down this process:

1. **Reading the ELF Header**: An ELF binary starts with a header that contains important information about the structure of the file. This header includes details about the different sections of the binary, such as the text section (with contains the executable code) and the data section (which holds certain variables). The OS begins by reading this ELF header to understand how the binary is organized.

2. **Mapping Sections into Virtual Memory**: After reading the ELF header, the OS maps the various sections of the binary into the process’s virtual memory. This means that the OS allocates memory for each section and sets up the process’s virtual address space according to the instructions provided by the ELF file. For example:

   - The **text section** is mapped to a region of memory where the CPU can execute the code.
   - The **data section** is mapped to a separate region where variables and other data will be stored.

3. **Setting Up the Process Environment**: Once the sections are mapped into memory, the OS sets up the process’s environment. This includes initializing the stack, setting up the heap (for dynamic memory allocation), and preparing any necessary environment variables and command-line arguments.

4. **Starting the Process**: Finally, the OS transfers control to the entry point of the program, which is typically the `main()` function. At this point, the program begins executing its instructions.

Tools like `readelf` and `objdump` are useful for exploring the structure of ELF binaries. They allow you to inspect the ELF header, view the different sections, and understand how the binary is laid out in memory.

#### Permissions and Setuid Binaries

Earlier, we discussed how a buffer overflow can be used to modify variables on the stack. But this raises an important question: why can’t we read "flag.txt" directly from the command line, but the exploited binary can? The answer lies in file permissions.

**File permissions** control who can read, write, or execute a file. You can view file permissions from the command line using the `ls -l` command, which shows a string of characters indicating the permissions for the owner, group, and others. For example, `rwxr-xr-x` means the owner can read, write, and execute the file, while the group and others can only read and execute it.

Permissions also apply to binaries. When you run the `stack0` binary, you might notice that it’s a "setuid" ELF executable. The `setuid` bit is a special permission that allows a binary to run with the privileges of the file’s _owner_, rather than the user who started it. This is why the binary can read "flag.txt" even though you can’t directly.

The `setuid` feature is useful for tasks that require temporary elevated privileges, like accessing hardware or managing system resources. For example, commands like `ping` and `sudo` are setuid binaries. However, the OS has safeguards to prevent misuse, ensuring that an unprivileged user can’t simply write a program and use `setuid` to run it as root.

The `setuid` feature is helpful for tasks that require temporary elevated privileges, like accessing hardware or managing system resources. For example, commands like `ping` and `sudo` are setuid binaries. Fortunately, the OS has safeguards to prevent misuse, ensuring that an unprivileged user can’t simply write a program and use `setuid` to run it as root.

### PIE Binaries

Some binaries are compiled to be position-independent, meaning that their code sections can be loaded at different memory locations each time the program runs. These are called **Position Independent Executables** or **PIE** binaries. PIE, when combined with Address Space Layout Randomization (ASLR), enhances security by making it harder for attackers to predict where specific functions or code sequences will be located in memory. This feature complicates the creation of exploits because, as an attacker, you often need to know the exact memory addresses of certain functions. We'll talk more about randomization and how to bypass it in future lectures.

You can check if a binary is compiled with PIE support using the `checksec` utility, which is included in the EpicTreasure docker image. For example:

```
$ checksec ./stack2-64
[*] '/root/host-share/stack2-64'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

Another clue that PIE is enabled is if function addresses are at low addresses, as shown in this example:

```
$ gdb ./stack2-64
+pwndbg> p win
$1 = {void ()} 0x7aa <win>
```

Here, the `win()` function is located at `0x7aa`, a low address, indicating that PIE is in use. To find the actual address of `win()` at runtime, you can use GDB and set a breakpoint:

```
$ gdb ./stack2-64
+pwndbg> b main
Breakpoint 1 at 0x824: file stack2.c, line 24.
+pwndbg> r

...Omitted for clarity...

+pwndbg> p win
$2 = {void ()} 0x5555555547aa <w

```

Notice that the last three nibbles (a nibble is half of a byte, or 4 bits) of the address `0x7aa` are the same as in the previous output. This isn’t a coincidence---it indicates that `0x7aa` is an offset from the start of the text section. You can use the `info proc map` command in GDB to find out where the text section starts, as well as the locations and sizes of other regions in memory.

_Psst. Check the source, but be quiet. Don't let the Captain or Professor-Ambassador Walls
see you._

<!-- Are you sure you weren't followed? We haven't met yet, but I've heard of you. I'm Ensign Scar'dy.  There seems to be something strange going on. I found this flag lying around, `DoTheRequiredReading`, but I'm not sure where it belongs. I think Walls is up to something. Maybe he left some clues? He's crafty, so he probably put those clues in a place you're supposed to look...-->

```
$ gdb ./stack2-64
+pwndbg> b main
Breakpoint 1 at 0x824: file stack2.c, line 24.
+pwndbg> r

...Omitted for clarity...

+pwndbg> info proc map
process 2155
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
      0x555555554000     0x555555555000     0x1000        0x0 /root/host-share/stack2-64
...Omitted for clarity...
      0x7ffffffde000     0x7ffffffff000    0x21000        0x0 [stack]
...Omitted for clarity...

```

### Summary and Review

Today, we dove into the basics of buffer overflows, starting with a straightforward example. We explored how memory is organized on the stack and how understanding this layout can help us craft clever exploits. By now, you should feel more confident in spotting and exploiting buffer overflow vulnerabilities and understand how these exploits can lead to major security issues. We’ve also explored several fundamental concepts related to understanding and working with binaries on Linux, such as:

- [Virtual Address Space](#virtual-address-space-of-a-process)
- [Memory Layout](#memory-regions-in-a-process)
- [Buffer Overflows](#exploiting-the-buffer-overflow-in-stack0)
- [Disassembling](#looking-at-the-disassembly)
- [Return Addresses](#introducing-return-addresses)
- [ELF](<#elf-(executable-and-linkable-format)>)
- [Positional Independent Executables (PIE)](#pie-binaries)

As you review this material, take some time to review the challenge binaries and see how everything fits together. If you have any questions or need a bit more clarity, don’t hesitate to bring them up in lecture or during office hours.
