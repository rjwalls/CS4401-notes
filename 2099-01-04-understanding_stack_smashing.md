---
title:  "Lecture Notes: Understanding Smashing the Stack for Fun and Profit"
date:   2020-04-01 01:00:00
categories: notes lecture
layout: post
---

Previously, I assigned the classic "Smashing the Stack for Fun and Profit" as
your reading. Here, I want to clarify a few points from the first few sections
of the reading, i.e., the sections prior to the shellcode discussion. I will
post a separate set of notes specifically for shellcoding. 

#### Defining Buffers

First, the way the author defines a buffer is different from how I defined it
in the last lecture. He defines a buffer as a "contiguous block of computer
memory that holds multiple instances of the same data type", whereas I
typically define a buffer more as a "contiguous block of memory associated with
a data type." The difference here is minor, but my definition is more broad in
that it allows us to refer to the four bytes associated with a single integer as
a buffer also. 

#### Memory Permissions

Second, the author talks about about the code region of memory being marked as
read-only. Many of you have probably come across this idea before in past
classes (e.g., an undergraduate operating systems course). Broadly, we can
think of virtual memory simply as a long contiguous sequences of bytes where
each byte is associated with an address. Each address (through the magic of
paging, again see your favorite OS textbook for details) is also associated
with a set of permissions that specify whether that memory location can be read
from, written to, or executed. These permissions are analogous to the read,
write, and execute permissions that you see associated with files and file
systems.  If you encounter the dreaded **segmentation fault** when trying to
solve a challenge binary, then understand that the executing process tried to
violate these permissions. 


#### Overwriting the Code Region

An understanding of memory permissions  also helps us answer another question
you might be thinking: "if the process's code is just sitting in memory, can an
attacker use a buffer overflow to change that code? The answer is usually "no"
here for a variety of reasons. First, the code typically lives in a memory
region that is marked as read-only.  Second, our stack-based buffer overflow
can only change memory that is *above* the buffer, and the code region
typically lies *below* the stack. Of course, you'll soon learn about a more
powerful form of memory corruption, often referred a **write-what-where**
vulnerability, that we can leverage to write to any address in memory (modulo
read-only permissions and unmapped memory). Further, in many cases, if we have
a write-what-where vulnerability *we can manipulate the program to such
a degree that we can change memory permissions.*  

#### Code Injection

If, as an attacker, you can't overwrite/modify the existing code for a process,
then the logical next step to supply your own malicious code. This class of
attack is generally referred to as **code injection**. That is what the "Shell
Code" section of smashing the stack is about.

A successful attack requires more than just inserting malicious code
(once again, via  external input, not by modifying the binary), you also have
to trick the process into executing that code. More on this in a bit. 

Later on, we will talk about how you can use memory permissions to protect
against code injection.

### Stack Frames

Third, the author talks about the stack frame, a concept that we skirted around
previously. Stack frames are not too complicated, we can define them as the
section of the stack associated with a single function call. Every time a
function is called, a new stack frame is created. Setting up that stack frame
involves a number of steps, some of which are the responsibility  of the caller
and some are the responsibility of the callee. This behavior is defined by the
**calling convention** and varies from system to system.  Importantly, the
instructions in each function responsible for stack frame setup and tear-down
are referred to as the **function prologue** and **epilogue**, respectively. 


### Spilling Registers

The prologue and epilogue are also used to setup registers for use. Remember
that there are only a small number of registers that a function can use, and
all functions use those registers. Once again this depends on the architecture,
this means that a function may have to save the previous value of a register
before it can be used, i.e., save and restore the value so when the previous
function executes the register still contains the value that it expects.

Where are these registers saved? On the stack of course. You may hear this
process referred to as **register spilling**. Have we seen this behavior in the
programs we've looked at so far? Yep, we've seen register spilling used for the
`rbp` register. Register spilling is another reason why the return address by
maybe further away from the start of the buffer than you would think based on
the C code.
 

### Outside of Address Space

Aleph One makes a confusing statement when talking about the segmentation fault
caused by the buffer overflow. Specifically, the address "0x41414141" (the hex
representation of ascii "AAAA") is "outside of the process address space" and
that is why your receive a segmentation fault. Last class, However, the virtual
address space on a 32-bit system spans addresses from 0x0 to 0xFFFFFFFF (i.e., all
addresses possible with a 32-bit address in a byte-addressable system). The
"AAAA" address certainly falls in that range. 

It is more accurate to say that the memory at address "AAAA" is not mapped. In
other words, the OS has not assigned any physical memory to process that is
associated with (or mapped to) that virtual address. Remember, the virtual
address space is an abstraction/illusion provided to the process by the OS.
Because the amount of physical memory is very limited (especially when you
consider that your machine may be running thousands of processes concurrently),
the OS only maps a small fraction of the process's address space. 

To complicate matters further, the kernel reserves a portion of the address
space for itself.  On 32-bit x86 Linux systems, it is common for the first 3
GiB addresses  (3 times 2^30 addresses) to be usable by the process and the
last 1 GiB to be usable only by the OS. The cutoff address would then be
0xc0000000. 

In GDB you can view the mapped memory using `info proc map`. 

```
gdb /opt/protostar/bin/stack0
break *main
r
info proc map
```

Here we can see the memory regions of the process. Notably, we have the code
region at the bottom and the stack at the top. We will ignore the rest of the
entries in this list for now. If you are looking closely, you might notice
something strange: previously I said the environment variables were at the top
of memory, but GDB tells us that the stack is pushed right up against the
boundary for kernel-only memory addresses (0xc0000000). This should tell you
that the environment variables actually live on the stack. 

