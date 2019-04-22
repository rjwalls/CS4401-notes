---
title:  "Lecture Notes: peda for gdb"
date:   2019-04-19 14:00:00
categories: notes
layout: post
---

# PEDA - Python Exploit Development Assistance for GDB


PEDA enhances the display of gdb to display more information like registers, 
stack memory, and dissassembled code. PEDA has a variety of built-in commands 
that are useful for exploiting binaries (creating shellcode, ROPgadgets, 
assembling code into hexidecimal, etc.)

To install PEDA:
```
git clone https://github.com/longld/peda.git ~/peda
echo "source ~/peda/peda.py" >> ~/.gdbinit
echo "DONE! debug your program with gdb and enjoy"
```

### PEDA's debugging environment

Once PEDA is installed in gdb, start gdb normally eg. `gdb ./stack1`. When 
you run a program, PEDA will display three sections: ***registers***, ***code***, and 
***stack***.
```
gdb-peda$ b main
Breakpoint 1 at 0x804858c: file stack1.c, line 13.
gdb-peda$ r
Starting program: /problems/stack1r_50_816d98b45f215d0f6c3b7850ee79a32d/stack1 

[----------------------------------registers-----------------------------------]
EAX: 0xf7fc6dbc --> 0xffffd66c --> 0xffffd7cc ("SHELL=/bin/bash")
EBX: 0x0 
ECX: 0xffffd5d0 --> 0x1 
EDX: 0xffffd5f4 --> 0x0 
ESI: 0xf7fc5000 --> 0x1afdb0 
EDI: 0xf7fc5000 --> 0x1afdb0 
EBP: 0xffffd5b8 --> 0x0 
ESP: 0xffffd560 --> 0x0 
EIP: 0x804858c (<main+17>:	sub    esp,0xc)
EFLAGS: 0x286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048586 <main+11>:	mov    ebp,esp
   0x8048588 <main+13>:	push   ecx
   0x8048589 <main+14>:	sub    esp,0x54
=> 0x804858c <main+17>:	sub    esp,0xc
   0x804858f <main+20>:	push   0x80486e0
   0x8048594 <main+25>:	call   0x8048410 <getenv@plt>
   0x8048599 <main+30>:	add    esp,0x10
   0x804859c <main+33>:	mov    DWORD PTR [ebp-0xc],eax
[------------------------------------stack-------------------------------------]
0000| 0xffffd560 --> 0x0 
0004| 0xffffd564 --> 0xffffd604 --> 0x13bb2be3 
0008| 0xffffd568 --> 0xf7fc5000 --> 0x1afdb0 
0012| 0xffffd56c --> 0xa287 
0016| 0xffffd570 --> 0xffffffff 
0020| 0xffffd574 --> 0x2f ('/')
0024| 0xffffd578 --> 0xf7e21dc8 --> 0x2b76 ('v+')
0028| 0xffffd57c --> 0xf7fd41a8 --> 0xf7e15000 --> 0x464c457f 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, main (argc=0x1, argv=0xffffd664) at stack1.c:13
13	  variable = getenv("GREENIE");
gdb-peda$ 
```
The fist section ***registers*** displays the registers and their values.  PEDA 
is smart and recognizes when a pointer is stored in memory and shows you the 
value that it points to. This is useful when you need to track your input string 
in memory.
```
[----------------------------------registers-----------------------------------]
EAX: 0xf7fc6dbc --> 0xffffd66c --> 0xffffd7cc ("SHELL=/bin/bash")
EBX: 0x0 
ECX: 0xffffd5d0 --> 0x1 
EDX: 0xffffd5f4 --> 0x0 
ESI: 0xf7fc5000 --> 0x1afdb0 
EDI: 0xf7fc5000 --> 0x1afdb0 
EBP: 0xffffd5b8 --> 0x0 
ESP: 0xffffd560 --> 0x0 
EIP: 0x804858c (<main+17>:	sub    esp,0xc)
EFLAGS: 0x286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[------------------------------------------------------------------------------]
```
The ***code*** section displays the assembly code that it's executing.  There 
is an arrow that points to the exact assembly instruction that is about to be 
executed. In this example it's `sub  esp,0xc` at location `0x804858c` in 
memory. Using the command `ni`, you can step through the instructions one by 
one.
```
[-------------------------------------code-------------------------------------]
   0x8048586 <main+11>:	mov    ebp,esp
   0x8048588 <main+13>:	push   ecx
   0x8048589 <main+14>:	sub    esp,0x54
=> 0x804858c <main+17>:	sub    esp,0xc
   0x804858f <main+20>:	push   0x80486e0
   0x8048594 <main+25>:	call   0x8048410 <getenv@plt>
   0x8048599 <main+30>:	add    esp,0x10
   0x804859c <main+33>:	mov    DWORD PTR [ebp-0xc],eax
[------------------------------------------------------------------------------]
```
Last, the ***stack*** section displays the contents of the stack, and shows what 
addresses the stack is using. Here, the top of the stack is at `0xffffd560`. If 
we look at the left side of the display, it shows the byte offset for each line. 
Since the bytes are increasing by 4 for each line, PEDA is displaying **words** on 
the stack.  This means that `0x00000000` or `0x0` is the first word, and `0xffffd604` is the 
second. PEDA knows that the second word is a pointer so it displays what it is pointing 
to: `0xd2ae39cc`.
```
[------------------------------------stack-------------------------------------]
0000| 0xffffd560 --> 0x0 
0004| 0xffffd564 --> 0xffffd604 --> 0xd2ae39cc 
0008| 0xffffd568 --> 0xf7fc5000 --> 0x1afdb0 
0012| 0xffffd56c --> 0xa287 
0016| 0xffffd570 --> 0xffffffff 
0020| 0xffffd574 --> 0x2f ('/')
0024| 0xffffd578 --> 0xf7e21dc8 --> 0x2b76 ('v+')
0028| 0xffffd57c --> 0xf7fd41a8 --> 0xf7e15000 --> 0x464c457f 
[------------------------------------------------------------------------------]
```

### PEDA's helpful commands

1. `vmmap` - Get virtual mapping address ranges of section(s) in debugged process
```
gdb-peda$ vmmap
Start      End        Perm	Name
0x08048000 0x08049000 r-xp	/problems/heap1r_20_013443d53348cfdbfb9fb6492286d09f/heap1
0x08049000 0x0804a000 r--p	/problems/heap1r_20_013443d53348cfdbfb9fb6492286d09f/heap1
0x0804a000 0x0804b000 rw-p	/problems/heap1r_20_013443d53348cfdbfb9fb6492286d09f/heap1
0x0804b000 0x0806c000 rw-p	[heap]
0xf7e14000 0xf7e15000 rw-p	mapped
0xf7e15000 0xf7fc2000 r-xp	/lib32/libc-2.23.so
0xf7fc2000 0xf7fc3000 ---p	/lib32/libc-2.23.so
0xf7fc3000 0xf7fc5000 r--p	/lib32/libc-2.23.so
0xf7fc5000 0xf7fc6000 rw-p	/lib32/libc-2.23.so
0xf7fc6000 0xf7fc9000 rw-p	mapped
0xf7fd4000 0xf7fd5000 rw-p	mapped
0xf7fd5000 0xf7fd8000 r--p	[vvar]
0xf7fd8000 0xf7fd9000 r-xp	[vdso]
0xf7fd9000 0xf7ffc000 r-xp	/lib32/ld-2.23.so
0xf7ffc000 0xf7ffd000 r--p	/lib32/ld-2.23.so
0xf7ffd000 0xf7ffe000 rw-p	/lib32/ld-2.23.so
0xfffdd000 0xffffe000 rw-p	[stack]
```
2. `assemble` - Convert assembly code to hexidecimal instructions
```
gdb-peda$ assemble
Instructions will be written to stdout
Type instructions (NASM syntax), one or more per line separated by ";"
End with a line saying just "end"
iasm|0xdeadbeef> pop eax            
hexify: "\x58"
iasm|0xdeadbef0> ret
hexify: "\xc3"
iasm|0xdeadbef1> end
Assembled instructions:
"\x58"  # 0x00000000:    pop eax
"\xc3"  # 0x00000001:    ret

hexify: "\x58\xc3"
```
3. `find` - Find a string of characters in a page or between two addresses
```
gdb-peda$ find “/bin/sh” libc
Searching for '/bin/sh' in: libc ranges
Found 1 results, display max 1 items:
libc : 0xf7f6e02b ("/bin/sh")
```
4. `shellcode` - Generate common shellcode
```
gdb-peda$ shellcode generate x86/linux exec
# x86/linux/exec: 24 bytes
shellcode = (
    "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31"
    "\xc9\x89\xca\x6a\x0b\x58\xcd\x80"
)
```
5. `ropsearch` - Search for ROP gadgets
```
gdb-peda$ ropsearch "pop eax" libc
Searching for ROP gadget: 'pop eax' in: libc ranges
0xf7e38f97 : (b'58c3')	pop eax; ret
0xf7ee8caa : (b'58c3')	pop eax; ret
0xf7efd6b2 : (b'58c3')	pop eax; ret
0xf7f31ef6 : (b'58c3')	pop eax; ret
0xf7f7dd71 : (b'58c3')	pop eax; ret
0xf7efd691 : (b'58c3')	pop eax; ret
0xf7f7adfc : (b'58c3')	pop eax; ret
0xf7e82856 : (b'5810e9c3')	pop eax; adc cl,ch; ret
0xf7ed7d59 : (b'5804f7c3')	pop eax; add al,0xf7; ret
...
```
6. `aslr` - Check ASLR setting for GDB
```
gdb-peda$ aslr
ASLR is OFF
```



For more resources:
1. https://github.com/longld/peda
2. https://github.com/ebtaleb/peda_cheatsheet/blob/master/peda.md

Author:
Marc Reardon
April 19, 2019
