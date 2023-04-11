---
title: "Calling Conventions"
date: 2020-01-02 09:01:01
categories: notes lecture
layout: post
---

## What is a calling convention?
A calling convention is a set of rules that dictate how a program should call a function. Programming languages like C abstract away low-level functionality present in binaries. The compiler decides for us how to move the stack pointer and pass parameters into functions in machine code. These decisions are specified by an **Application Binary Interface** (ABI). The parts of the ABI that specify how function calls should work is the calling convention.

Most Linux (or GNU/Linux if you're pedantic) programs use the **System V ABI** (pronounced "System Five"). The System V ABI consists of a Generic ABI document and Processor Supplement documents that contain hardware-specific standards. These documents are available at https://refspecs.linuxbase.org/.

For this course, we're interested in Intel386 and x86_64, which apply to the 32- and 64-bit challenges, respectively.
To illustrate how calling conventions work, I've written a C program where `main` calls a function that adds six numbers together and returns the sum. 

Below are annotated diagrams for both binary types. The source code is identical for both, but the compiled assembly code is different, highlighting how the parameters in the function call are passed as per the calling convention.

## 64-bit
The x86_64 System V ABI specifies that function parameters are passed via registers in this order: `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`. If a function has more than 6 parameters, the rest go on the stack. The return value is stored in `rax`. 

<img src="https://raw.githubusercontent.com/rjwalls/CS4401-notes/master/assets/calling-conventions/cc64.png" alt="64 bit calling conventions diagram" width="700"/>

***Note**: The System V ABI defines both `rax` and `rdx` as return registers. Since C functions only return one value, `rdx` is only used for this purpose when the return type is larger than 64 bits. For example, GCC provides a nonstandard `unsigned __int128` type. Both registers will be used for a function that returns that 128-bit type.*

***Another note**: You might notice that `r9d`, `r8d`, and `e` versions of the last four registers are used instead of the normal `r` registers. These represent the lower 32 bits of the 64-bit `r` registers. They're used because the parameters we're passing are not large enough to require the full registers.*

## 32-bit
The i386 System V ABI specifies that function parameters are passed on the stack, and the single return register is `eax`. Note the order of the parameters: They are pushed onto the stack in *reverse* order, i.e. the last parameter first. This means they end up ordered in memory as they are in the C code.
<img src="https://raw.githubusercontent.com/rjwalls/CS4401-notes/master/assets/calling-conventions/cc32.png" alt="32 bit calling conventions diagram" width="700"/>