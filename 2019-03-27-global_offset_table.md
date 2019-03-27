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
`ldd sample` multiple times. How can our simple binary now these addresses for
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
puts in the procedure linkage table (plt or PLT) at address `0x400400`. When
you compile your binary there are sections called relocations which are left
for the linker to fill in at runtime [6]. The linker determines some value,
maybe an address, and places that value inside the binary at some offset [6]. You
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


### Sources

1. https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html
2. https://www.youtube.com/watch?v=kUk5pw4w0h4
3. https://stackoverflow.com/questions/20486524/what-is-the-purpose-of-the-procedure-linkage-table
4. https://www.youtube.com/watch?v=t1LH9D5cuK4
5. https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html
6. 
