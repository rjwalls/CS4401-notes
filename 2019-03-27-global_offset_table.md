---
title: "Global Offset Table"
date: 2019-03-27 09:00:00
categories: information 
layout: post
---

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
screen. Nice and easy. However, this dinamically-linked binary calls the
`printf` function from the standard C library. We can check this out by running
`ldd sample` where `sample` is the binary compiled with the C code shown above.
We get the result

```
linux-vdso.so.1 =>  (0x00007ffff7ffa000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff7a0d000)
/lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd7000)
```

As you can see this are the base addresses of the libraries called. Note that
these addresses will change due to ASLR. You can see by yourself by running
`ldd sample` multiple times. How can our simple binary know these addresses for
the necessary libraries if they change? To find out, first lets disassemble our
binary by running `gdb sample`. Lets set the syntax to being intel style by
running `set disassembly-flavor intel`. Run `disas main` and we get

```assembly
Dump of assembler code for function main:
   0x0000000000400526 <+0>:	push   rbp
   0x0000000000400527 <+1>:	mov    rbp,rsp
   0x000000000040052a <+4>:	sub    rsp,0x10
   0x000000000040052e <+8>:	mov    DWORD PTR [rbp-0x4],edi
   0x0000000000400531 <+11>:	mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000400535 <+15>:	mov    edi,0x4005d4
   0x000000000040053a <+20>:	call   0x400400 <puts@plt>
   0x000000000040053f <+25>:	mov    eax,0x0
   0x0000000000400544 <+30>:	leave
   0x0000000000400545 <+31>:	ret
End of assembler dump.
```

We see values loaded into registers and whatnot but what we're really
interested in is this line

```assembly
  0x000000000040053a <+20>:	call   0x400400 <puts@plt>
```

Here is where our external function gets called and how we're able to obtain
its address to call it. One minor note, it does say `puts@plt` instead of
`printf@plt` because of a compiler/linker optimization. It does not affect the
execution of our binary nor the point we're making in this post. Thus we call
`puts` in the procedure linkage table (plt or PLT) at address `0x400400`. When
you compile your binary there are sections called relocations which are left
for the linker to fill in at runtime [5]. The linker determines some value,
maybe an address, and places that value inside the binary at some offset [5]. You
could look at the relocations the compiler leaves behind by running `gcc -c
sample.c` to compile and `readelf --relocs ./sample.o` which gives us

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
   0x0000000000400400 <+0>:	jmpq   *0x200c12(%rip)        # 0x601018
   0x0000000000400406 <+6>:	pushq  $0x0
   0x000000000040040b <+11>:	jmpq   0x4003f0
End of assembler dump.
```

Interesting, so we jumped to the PLT but now from there we are jumping somwhere
else. Specifically to this address `0x601018`. We can see what's there. We run
`disas 0x601018` and we get

```assembly
Dump of assembler code for function _GLOBAL_OFFSET_TABLE_:
   0x0000000000601000:	sub    %cl,(%rsi)
   0x0000000000601002:	(bad)
   0x0000000000601003:	add    %al,(%rax)
   0x0000000000601005:	add    %al,(%rax)
   0x0000000000601007:	add    %al,(%rax)
   0x0000000000601009:	add    %al,(%rax)
   0x000000000060100b:	add    %al,(%rax)
   0x000000000060100d:	add    %al,(%rax)
   0x000000000060100f:	add    %al,(%rax)
   0x0000000000601011:	add    %al,(%rax)
   0x0000000000601013:	add    %al,(%rax)
   0x0000000000601015:	add    %al,(%rax)
   0x0000000000601017:	add    %al,(%rsi)
   0x0000000000601019:	add    $0x40,%al
   0x000000000060101b:	add    %al,(%rax)
   0x000000000060101d:	add    %al,(%rax)
   0x000000000060101f:	add    %dl,(%rsi)
   0x0000000000601021:	add    $0x40,%al
   0x0000000000601023:	add    %al,(%rax)
   0x0000000000601025:	add    %al,(%rax)
   0x0000000000601027:	add    %al,(%rax)
End of assembler dump.
```

We have trampolined to the global offset table (got or GOT). But what is
exactly happening here? It is called lazy binding. The dynamic linker loads
a library and places an identifier in the GOT. When the function gets called we
jump to a default stub from the PLT into the GOT (`0x0000000000400400 <+0>:	jmpq   *0x200c12(%rip)        # 0x601018`)
then the identifier is loaded (`0x0000000000400406 <+6>:	pushq  $0x0`) and the
dynamic linker is called. Now there is enough information for the linker to
patch the address in the GOT of the library function we wanted to call in our
binary. We will then jump to this address from the PLT (`0x000000000040040b <+11>:	jmpq   0x4003f0`)
rather than our default stub. As an illustration we set a breakpoint right after the
call to `puts` has been done and run the binary with gdb. We look again in the
global offset table by doing `disas 0x601018` and we see this

```assembly
Dump of assembler code for function _GLOBAL_OFFSET_TABLE_:
   0x0000000000601000:	sub    %cl,(%rsi)
   0x0000000000601002:	(bad)
   0x0000000000601003:	add    %al,(%rax)
   0x0000000000601005:	add    %al,(%rax)
   0x0000000000601007:	add    %ch,-0x1f(%rax)
   0x000000000060100a:	push   %rdi
   0x000000000060100c:	(bad)
   0x000000000060100d:	jg     0x60100f
   0x000000000060100f:	add    %dl,(%rax)
   0x0000000000601011:	out    %al,(%dx)
   0x0000000000601012:	fdivp  %st,%st(7)
   0x0000000000601014:	(bad)
   0x0000000000601015:	jg     0x601017
   0x0000000000601017:	add    %dl,-0x8583a(%rax)
   0x000000000060101d:	jg     0x60101f
   0x000000000060101f:	add    %al,-0x29(%rax)
   0x0000000000601022:	movabs %al,0x7ffff7
End of assembler dump.
```

It has been changed just like we expected to happen. 

### TL;DR

1. Compiler leaves spaces called relocations for linker to fill them in with
   addresses of functions from shared libraries.
2. Libraries change locations in memory due to ASLR.
3. We call library function from our C code. We reach the PLT and from the PLT
   we reach a stub entry of the global offset table and push an identifier.
4. This signals linker to place address of library function in GOT. We jump
   again from PLT to GOT and get the address of the library function.
5. Success, we can call the function.

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
is located at. Lets overwrite that using gdb. So we do `set
{int}0x804a01c=0x804853b` and we do `c` for continue. Surprise surpise, the
program outputs

`
Oh, it is on, like a prawn who yawns at dawn
`

we have successfully redirected code execution by overwriting the address on
the global offset table.

### Sources

1. https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html
2. https://www.youtube.com/watch?v=kUk5pw4w0h4
3. https://stackoverflow.com/questions/20486524/what-is-the-purpose-of-the-procedure-linkage-table
4. https://www.youtube.com/watch?v=t1LH9D5cuK4
5. https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html
