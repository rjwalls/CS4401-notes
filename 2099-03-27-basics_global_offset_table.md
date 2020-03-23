---
title: "Lecture Notes: Basics of the Global Offset Table"
date: 2019-03-27 09:00:00
categories: notes lecture
layout: post
challenges: format2 heap3 format4
---

So far, we've mostly targeted return addresses on the stack for our exploits with the occasional manipulation of a function pointer. However, there are many other locations we can target to hi-jack the program's control flow. In this set of notes, we describe the **Global Offset Table** a particularly useful structure that contains plenty of interesting code pointers for us to manipulate.


There exists two types of binaries: statically-linked and dynamically-linked.
The first kind is self-contained. Meaning it has all the functions and code
needed to execute. These binaries do not use external libraries. On the other
hand dynamically-linked binaries depend on system libraries to execute [1]. For
instance consider the following C code:

```c
#include <stdio.h>

int main(int argc, char **argv){

  printf("Oh, it is on, like a prawn who yawns at dawn.\n");

  return 0;

}
```

After compiling and executing our binary we get a string displayed on our
screen. Nice and easy. However, this dynamically-linked binary calls the
`printf` function from the standard C library. We can get the list of
dynamically linked binaries used by our program by running `ldd sample`---where
`sample` is the binary compiled with the C code shown above.  We get the result

```
linux-vdso.so.1 =>  (0x00007ffff7ffa000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff7a0d000)
/lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd7000)
```

As you can see these are the base addresses of the libraries called. Note that
these addresses will change due to ASLR. You can see by yourself by running
`ldd sample` multiple times. How can our simple binary know these addresses for
the necessary libraries if they change? To find out, first lets disassemble our
binary by running `gdb sample`. Lets set the syntax to being intel style by
running `set disassembly-flavor intel`. Run `disas main` and we get

```assembly
Dump of assembler code for function main:
   0x0000000000400526 <+0>:  push  rbp
   0x0000000000400527 <+1>:  mov   rbp,rsp
   0x000000000040052a <+4>:  sub   rsp,0x10
   0x000000000040052e <+8>:  mov   DWORD PTR [rbp-0x4],edi
   0x0000000000400531 <+11>: mov   QWORD PTR [rbp-0x10],rsi
   0x0000000000400535 <+15>: mov   edi,0x4005d4
   0x000000000040053a <+20>: call  0x400400 <puts@plt>
   0x000000000040053f <+25>: mov   eax,0x0
   0x0000000000400544 <+30>: leave
   0x0000000000400545 <+31>: ret
End of assembler dump.
```

We see values loaded into registers and whatnot but what we're really
interested in is this line

```assembly
  0x000000000040053a <+20>:	call   0x400400 <puts@plt>
```

Here is where our external function gets called and how we're able to obtain
its address to call it. One minor note, the binary uses `puts@plt` instead of
`printf@plt` because of a compiler/linker optimization. Namely, the compiler
sees that we are using a static string without a format specifier and decides
that `printf` is not needed. However, the compiler's decision does not change
the intended functionality of our binary nor the point we're making in this
post. 

Interpreting this line,  the program will call `puts` in the procedure linkage
table (plt or PLT) at address `0x400400`.  When you compile your binary there
are sections called **relocations** which are left for the linker to fill in at
runtime [5]. The linker determines some value, maybe an address, and places
that value inside the binary at some offset [5]. You could look at the
relocations the compiler leaves behind by running `gcc -c sample.c` to compile
and `readelf --relocs ./sample.o` which gives us

```
Relocation section '.rela.text' at offset 0x208 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000010  00050000000a R_X86_64_32       0000000000000000 .rodata + 0
000000000015  000a00000002 R_X86_64_PC32     0000000000000000 puts - 4

Relocation section '.rela.eh_frame' at offset 0x238 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```

As you can see there's a relocation for the `puts` function so we need the
address of this function in libc and have the linker place it inside our
binary.

```
000000000015  000a00000002 R_X86_64_PC32     0000000000000000 puts - 4
```

 Now, the PLT as mentioned above helps find these addresses. But how does it do
that? Well, first lets examine the address `0x400400`, the one referenced when
calling `puts`. We run `disas 0x400400` and we get

```assembly
Dump of assembler code for function puts@plt:
   0x0000000000400400 <+0>:  jmp    QWORD PTR [rip+0x200c12]        # 0x601018
   0x0000000000400406 <+6>:  push   0x0
   0x000000000040040b <+11>: jmp    0x4003f0
End of assembler dump.
```

Interesting, so we jumped to the PLT but we then immediately jump somewhere
else, specifically to `0x601018`. This technique of
jumping to a location and then immediately jumping somewhere else based on a
stored value is often called **function trampolining**.

Actually, we have trampolined to an address stored in the **global offset table** (got or
GOT). We can see where the global offset table begins and what's stored at that address
by running `x _GLOBAL_OFFSET_TABLE_` which gives us `0x600e28:	0x00000001`. We
can see more of the GOT, including address `0x601018` by running `x/128wx _GLOBAL_OFFSET_TABLE_`
and we get

```
0x600e28:	0x00000001	0x00000000	0x00000001	0x00000000
0x600e38:	0x0000000c	0x00000000	0x004003c8	0x00000000
0x600e48:	0x0000000d	0x00000000	0x004005c4	0x00000000
0x600e58:	0x00000019	0x00000000	0x00600e10	0x00000000
0x600e68:	0x0000001b	0x00000000	0x00000008	0x00000000
0x600e78:	0x0000001a	0x00000000	0x00600e18	0x00000000
0x600e88:	0x0000001c	0x00000000	0x00000008	0x00000000
0x600e98:	0x6ffffef5	0x00000000	0x00400298	0x00000000
0x600ea8:	0x00000005	0x00000000	0x00400318	0x00000000
0x600eb8:	0x00000006	0x00000000	0x004002b8	0x00000000
0x600ec8:	0x0000000a	0x00000000	0x0000003d	0x00000000
0x600ed8:	0x0000000b	0x00000000	0x00000018	0x00000000
0x600ee8:	0x00000015	0x00000000	0x00000000	0x00000000
0x600ef8:	0x00000003	0x00000000	0x00601000	0x00000000
0x600f08:	0x00000002	0x00000000	0x00000030	0x00000000
0x600f18:	0x00000014	0x00000000	0x00000007	0x00000000
0x600f28:	0x00000017	0x00000000	0x00400398	0x00000000
0x600f38:	0x00000007	0x00000000	0x00400380	0x00000000
0x600f48:	0x00000008	0x00000000	0x00000018	0x00000000
0x600f58:	0x00000009	0x00000000	0x00000018	0x00000000
0x600f68:	0x6ffffffe	0x00000000	0x00400360	0x00000000
0x600f78:	0x6fffffff	0x00000000	0x00000001	0x00000000
0x600f88:	0x6ffffff0	0x00000000	0x00400356	0x00000000
0x600f98:	0x00000000	0x00000000	0x00000000	0x00000000
0x600fa8:	0x00000000	0x00000000	0x00000000	0x00000000
0x600fb8:	0x00000000	0x00000000	0x00000000	0x00000000
0x600fc8:	0x00000000	0x00000000	0x00000000	0x00000000
0x600fd8:	0x00000000	0x00000000	0x00000000	0x00000000
0x600fe8:	0x00000000	0x00000000	0x00000000	0x00000000
0x600ff8:	0x00000000	0x00000000	0x00600e28	0x00000000
0x601008:	0x00000000	0x00000000	0x00000000	0x00000000
0x601018:	0x00400406	0x00000000	0x00400416	0x00000000
```

Ideally, that stored address points to where `puts` has been loaded into
memory. But it turns out that things are more complicated, namely, the address
in the GOT is not updated until the program tries to call `puts` the first
time. This process is called **lazy binding**. 

The first time the program calls `puts`, the address in the GOT entry for
`puts` actually points to a special bit of code that results in the dynamic linker
loading the library and updating the GOT with the correct addresses. In future
calls to `puts`, the `jmp` in the PLT will directly go to `puts` (since the GOT
entry has been updated) rather than jumping to the linker code. 

More concretely, the first time the program calls `puts` the "special bit" of
code that's executed actually starts with the two instructions
instructions we saw in the PLT:  (`0x0000000000400406 <+6>:	push   0x0
`) tells
the linker that function `0`, i.e., `puts`, was called and the next instruction
jumps to the code to start the linking process (`0x000000000040040b <+11>:	jmp    0x4003f0`).

As an illustration, if we set a breakpoint right after the first call
to `puts`,  we see that the default values in the GOT have been replaced with
the proper addresses---because the linker has updated the
GOT to hold the correct address for `puts`. Next we check the where the GOT
begins by running `x _GLOBAL_OFFSET_TABLE_` gives back `0x601000:	0x00600e28`.
Interesting, the start of the GOT is now closer to address `0x601018` where the
address of `puts` is stored. If we run `x/8wx _GLOBAL_OFFSET_TABLE_` we get
back 

```
0x601000:	0x00600e28	0x00000000	0xf7ffe168	0x00007fff
0x601010:	0xf7deee10	0x00007fff	0xf7a7c690	0x00007fff
```

This means that the address for `puts` must be `0xf7a7c690`. We can confirm
this by runnning `x 0x601018` which should give us the same result.

### TL;DR

1. The program doesn't know at compile-time where dynamically linked libraries
   will be placed in memory, so it doesn't know exactly where functions like
`puts` will be placed.
2. As a result, the program will instead calls a stub function for `puts` at a
   location it does know, i.e., in the procedure linkage table. 
3. The stub function (`puts@plt`) will get the runtime address of the real
   `puts` from the global offset table (a data structure updated by the linker
at runtime).
4. When we first call the library function from our C code, the linker
   will update the GOT to use the correct address of the library function.

### Example

Consider the following C code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void hello()
{
  printf("Oh, it is on, like prawn who yawns at dawn\n");
  _exit(1);
}

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printf(buffer);

  exit(1);   
}

int main(int argc, char **argv)
{
  vuln();
}
```

After compiling this code, we open the binary in gdb. Our goal is to execute
the `hello` function by overwriting the global offset table. First we need the
address we will be using for the overwrite i.e. the address of the `hello`
function. We can do this in gdb by doing `x hello` and we get as a result
`0x804853b <hello>:	0x83e58955` so we save the address `0x804853b`. We `disas
main`. We see that `main` is calling another function we defined called `vuln`
so we set a breakpoint there. We `run`, we hit the breakpoint, and we step into
the function using the `si` command. We run `disas` to look at the assembly. We
see some library functions being called. Lets follow the last one library
function called into the plt and the got. So we see `0x080485d8 <+61>:	call
0x8048400 <exit@plt>`. Lets do `disas 0x080485d8`. We see 

```assembly
Dump of assembler code for function exit@plt:
   0x08048400 <+0>:	jmp    *0x804a01c
   0x08048406 <+6>:	push   $0x20
   0x0804840b <+11>:	jmp    0x80483b0
End of assembler dump.
```

We now know the address in the GOT where the address for the `exit` C function
is located. Lets overwrite that using gdb. So we do `set
{int}0x804a01c=0x804853b` and we do `c` for continue. Surprise surpise, the
program outputs

`
Oh, it is on, like a prawn who yawns at dawn
`

we have successfully redirected code execution by overwriting the address on
the global offset table.

### Sources

Check out the following links for more information on how the global offset table works and how you might use it when exploiting binaries. 

1. https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html
2. https://www.youtube.com/watch?v=kUk5pw4w0h4
3. https://stackoverflow.com/questions/20486524/what-is-the-purpose-of-the-procedure-linkage-table
4. https://www.youtube.com/watch?v=t1LH9D5cuK4
5. https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html
