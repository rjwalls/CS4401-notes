---
title:  "Lecture Notes: Memory Permissions, Register Spilling, and Other
Important Concepts"
date:   2020-01-04 02:00:00
categories: notes lecture
layout: post
---


Before you read the seminal article, "Smashing the Stack for Fun and Profit,"
we need to review some important concepts. Here I focus on the first few
sections of that classic article. I cover the shellcoding sections in a
separate set of notes.
 

#### Defining Buffers

The article defines a buffer differently than I did in our previous lectures.
Specifically, the article describes a buffer as a "contiguous block of computer
memory that holds multiple instances of the same data type." In contrast, I
typically define a buffer more as a contiguous block of memory associated with
a data type. The difference here is minor, but my definition is broader, e.g.,
it allows us to refer to the four bytes associated with a single integer as a
buffer.
 

#### Memory Permissions

Smashing the Stack" talks about the code region of memory as being marked as
read-only. Many of you have probably come across the concept of memory
permissions before in past classes, e.g., we cover this topic in *CS3013:
Operating Systems*. Broadly, the virtual address space is divided into
4096-byte blocks of memory called **pages**.  Each page is associated with a
set of permissions that specify whether the page can be read from, written to,
or executed. These permissions are analogous to the read, write, and execute
permissions that you see associated with files and file systems. Suppose you
encounter the dreaded segmentation fault when trying to solve a challenge
binary. In that case, that means the executing process tried to do something
that violated these permissions.

The concept of memory permissions also helps us answer another question you
might be wondering about.  Specifically, *if code for a process is just sitting
in memory, can an attacker use a buffer overflow to change that code?* The
answer is usually no for the following reasons. First, the code typically lives
in a read-only memory region. Any attempts to write to a read-only memory will
result in a segmentation fault. Second, our stack-based buffer overflows can
only directly change the memory at addresses higher than the buffer. The code
region on modern x86-based systems lies at lower memory addresses than the
stack. 

We will discuss more potent forms of memory corruption than buffer overflows in
a future lecture. Often referred to as **write-what-where vulnerabilities**, we
can leverage these bugs to write to any address in memory. Though writes to
read-only and unmapped memory pages still result in a segmentation fault.
However, in many cases, if we have a write-what-where vulnerability, we can
manipulate the program to such a degree that we can change memory permissions.


#### Code Injection

If, as an attacker, you can't overwrite/modify the existing code for a process,
then the logical next step to supply the malicious code. This class of attack
is generally referred to as **code injection**---the latter sections of the
"Smashing the Stack" article focus on this topic. We will talk more about code
injection in the next lecture.


#### Stack Frames

The article discusses stack frames. We can define a stack frame as the section
of the stack associated with a single function call. Every time a function is
called, a new stack frame is created. Setting up that stack frame involves
several steps, some of which are the caller's responsibility and some are the
callee's responsibility. The **calling convention** defines this behavior, and
it varies from system to system. Notably, each function's instructions for
stack setup and tear-down are referred to as the **function prologue** and
**epilogue**, respectively.

The prologue and epilogue also free up registers. You may remember from CS2011
that the x86-32/64 architectures offer a limited number of general-purpose
registers. All functions share those registers. Consequently, the callee
function has to save the previous value of a register before using that
register. This saving typically happens in the function prologue. Similarly,
the function epilogue includes instructions to restore the saved register
values. This process is referred to as **register spilling**.

Where are these registers saved/spilled? On the stack, of course. Have we seen
this behavior in the programs we've looked at so far? Yep, we've witnessed
register spilling used for the `rbp` registers. 
 

#### The Address Space

"Smashing the Stack" makes some confusing statements about the segmentation
fault caused by the buffer overflow. Specifically, the article states that the
address `0x41414141` (the hex representation of ascii `AAAA`) is "outside of
the process address space" and that is why the hardware raises a segmentation
fault. This statement is confusing because the virtual address space on a
32-bit system spans addresses from `0x0` to `0xFFFFFFFF`, i.e.,  this is the
range of all addresses possible with a 32-bit address in a byte-addressable
system. The address `0x41414141` certainly falls in that range.

It is more accurate to say that the memory at address `0x41414141` is unmapped.
In other words, the OS has not assigned (i.e., mapped) any physical memory to
that virtual address. As we previously discussed, the virtual address space is
an abstraction/illusion provided to the process. As the amount of physical
memory is limited, the OS/hardware only maps a small fraction of the process's
address space to physical memory.

Further, the kernel reserves a portion of the address space for itself. On
32-bit x86 Linux systems, it is common for the first 3 GiB addresses (3 times
2^30 addresses) to be usable by the process and the last 1 GiB to be reserved
for the OS. 

