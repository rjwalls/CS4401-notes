---
title:  "Lecture Notes: Basics of Buffer Overflows"
date:   2019-03-12 08:00:00
categories: notes lecture
layout: post
challenges: stack0 stack1 stack2 stack3 heap0 heap3
---


In this lecture, we are going to dive right into exploiting binaries. We will
begin with simple stack-based buffer overflows and work our way to the
venerable stack-smashing example. We try to answer important questions such as:
What is arbitrary code execution? What are `setuid` binaries?  What is
priviledge escalation? 


### Our First Challenge Binary

This the source code for the `stack0` challenge binary. Your objective is to
figure out how to exploit this binary. What do I mean by exploit? Well, loosely
I mean that your job is to take control of how `stack0` executes to accomplish
some nefarious goal. Here that goal is to read the contents of `flag.txt`;
presumably because we don't have permission to read it directly. 



```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  FILE *fp;

  modified = 0;
  gets(buffer);
  fp  = fopen("./flag.txt", "r");

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
      fgets(buffer, 64, (FILE*)fp);
      printf("flag: %s\n", buffer );
  } else {
      printf("Try again?\n");
  }
}
```
 

More generally, that goal involves gaining **arbitrary code execution** on the
target  machine. This means I (the attacker) can execute whatever code I want.
It should be straight-forward to see how arbitrary code execution can mean bad
things for the owner of the machine. 

How we (the attacker) obtain arbitrary code execution is one of the primary
subjects of this course; however, in this example the goal is actually much
simpler: we just want to modify the `modified` variable. 

Broadly, our recipe for success calls for two ingredients. First, the program
needs to contain a bug. Second, we need some way to supply input to the program
to trigger that bug. Let's focus on supplying the input for now.

There many different ways to supply input to a program. So how does this
example program take input? For many challenge binaries, the most obvious way
is to supply input via `stdin` (e.g., using `gets()`). But we can also provide
command line arguments and  environment variables. In other programs, external
input might come in the form of files on disk, network connections, external
temperature sensors, etc. 
 
What about the bug? What is the bug in this program? Let's take a closer look
at `gets`. Using `man gets` (on the target machine), we see that `gets()` reads
a line from `stdin` into a buffer...until either a terminating newline or
EOF...scrolling down to the BUGS section we see the following line of text:

> Never use gets(). Because it is impossible to tell...how many characters
> gets() will read. 

To really understand what this means we need to understand what memory looks
like and how variables are stored in that memory. We know that an array is is
just a sequence of contiguous bytes thrown somewhere in the giant abstract
block that we call **memory**. More generally, we can define a **buffer** as
any contiguous region of memory associated with a variable. So an integer might
be associated with buffer of size 4 bytes.[^buffdef]  

[^buffdef]: Some texts will define buffer slightly differently than the definition we use here. 

Now let's get back to `gets`. We know that `gets` in this example will copy
from `stdin` into the `buffer` character array, aka the buffer named `buffer`.
If we supply "aa" as input, then according to the man page, gets will store
three bytes: 'a', 'a', '\0'. But what happens if we supply 'a' times 100? It
will keep copying bytes and blow past the end of the array, i.e., write into
adjacent memory. 

This type of bug is called a **buffer overflow**. Now, machines have a lot of
memory, so what does in matter if a few bytes get overwritten?  Now we have to
consider what is next to this buffer in memory. Let's take a step back to fill
in some details.

*Virtual Address Space:* The operating system provides many wonderful
abstractions to processes executing on the machine. One of the most important
is related to memory. In particular, every process executing on a
machine---remember that a process is just an executing program---is given its
own virtual address space. From the perspective of the process, memory appears
to be a contiguous array of bytes that the process can use however it wants.
Each of those memory locations, can store a byte and is associated with an
address. 

This memory abstraction is great for the program because everything is simple,
for example, the process does not need to know the actual physical address of
the memory it is using. Further, the process is isolated from all other
processes; the memory for process A cannot be modified by process B. From now
on, when I say "memory" I am typically referring to the abstracted version of
memory provided by the OS, i.e., that contiguous stream of bytes.

By convention, memory (aka the virtual address space) is divided up into
different sections to be used for different purposes. Broadly, a process's
memory includes regions for the environment, the stack, the heap, data, and
text.   

The environment section includes the command line arguments, environment
variables, and similar information. The stack is what makes function calling
work (e.g., includes local variables, return values, etc.), the heap is used
for dynamically allocated memory (e.g., when using `malloc()`), and the text
section is primarily used for the compiled code. 

If we visualize this, we might see:

```
0xff ff ff ff
cmd env (set at process start)
stack (grows down, bottom is fixed)
heap (grows up)
Text (code, read only)
0x00 00 00 00
```

Here I have represented memory with the 0x0 address at the bottom and highest
address at the top. We can equivalently visualize memory in other orientations
(up, down, and sideways). Different text will use different orientations and
this will cause confusion.

It is also common to represent memory addresses using hex notation. Since our
challenge binary/VM is only 32-bit,[^32] that means we have a 32-bit virtual address
space and each address can be represented in 4 bytes (or 8 hex digits). 

[^32]: There are lots of ways we can determine if a binary targets a 32 or 64-bit architecture, but the easiest is probably the `file` command: `file bin_name`. 

Getting back to gets and our buffer overflow: gets will happily copy whatever
input we (aka the attacker) provide, even if that input is larger than the
buffer. This means our attacker can overwrite adjacent memory. So that is
adjacent to our buffer? In our challenge binary, the buffer is a local
variable. We know that local variables are stored on the stack. So the attacker
can overwrite information that is stored on the stack. Hmmm, the `modified`
variable---you know that variable we want to modify to solve the challenge---is
also on the stack. So let's see what happens. 

```bash

python -c "print 'a'*100;" | ./stack0

```

The above command will give us the response that we want: "you have changed the
'modified' variable". There is a big caveat here. We implicitly
assumed that the 'buffer' and 'modified' variables are adjacent to each other
in memory. Not only that, we assumed that the 'modified' variable is at a
higher address than 'buffer'. Our overflow wouldn't have given us the desired
results if the compiler swapped them in memory.  

Our assumption about the memory layout.
```
0xFF FF FF FF
...
modified
buffer
```

What could have happened:

```
0xFF FF FF FF
...
buffer
modified
```

Or even:

```
0xFF FF FF FF
...
modified
???
buffer
```

The `volatile` keyword is included here as it forces the compiler not to
optimize out the `modified` variable. This leads us to an important point:
**the compiler is going to transform the c code and we don't know what the
actually executed code will look like unless we examine the binary.** 

Another thing we will notice is that "segmentation fault" error. This means
that our actions have caused the program to attempt to use memory in a way that
isn't allowed. In this example, our input overwrote the modified variable and
then just kept on overwriting other values in memory. One of those other values
was the return address. 

To understand the concept of a return address, we need to
remember that process memory is also used for code. As we just discussed, 
the 'code' that is executed isn't the C code that we showed earlier. Instead,
that code is sequence of machine instructions. When a process is executing, the
CPU will iterate through this sequence of instructions and perform the
specified actions. We often visualize these machine instructions using assembly
language (even though there ASM is still a higher-level of abstraction).    

The x86 architecture uses a special register, called the instruction pointer,
to keep track of what instruction to execute next, i.e., it stores the memory
address of the next instruction. Think about what happens when a function call
occurs. The instruction pointer must changed to point to the function in
memory, but the old instruction point must also be saved so that the CPU can
return to where it was before. Given that there instruction pointer register
can only store a single address, we have to find some other place to save the
old pointer. The answer is to put it in memory, but where in memory? That's
another job for the stack.

Getting back to the seg fault, what is happening then? We overflow the buffer,
which overwrites `modified` and eventually overwrites the saved instruction
pointer (i.e., the return address). So when we return from the function, by
loading the old instruction pointer from the stack into our IP register, the
address is now just a bunch of 'aaaa' bytes. That address doesn't contain
anything, in fact it probably hasn't even be allocated so the OS freaks out and
kills the process. 

We can fix our exploit code by only overwriting the 'modified' variable. 


### Looking at the disassembly

We are going to use `gdb` to take a closer look at the instructions and memory
that comprise our challenge binary.[^wrong] The first thing we are going to go is
change how gdb displays assembly:

[^wrong]: Actually, this disassembly is for a different version of the code that is similar but does not read from `flag.txt`.


```
set disassembly-flavor intel
```

We do this purely based on personal preference. Now let's take a look at main:

```
(gdb) disassemble *main
0x080483f4 <main+0>:  push   ebp                      
0x080483f5 <main+1>:  mov    ebp,esp
0x080483f7 <main+3>:  and    esp,0xfffffff0           //done for aligment
0x080483fa <main+6>:  sub    esp,0x60                 //making space 
0x080483fd <main+9>:  mov    DWORD PTR [esp+0x5c],0x0 //initialize 'modified'
0x08048405 <main+17>: lea    eax,[esp+0x1c]           //load addr of buffer
0x08048409 <main+21>: mov    DWORD PTR [esp],eax      //set up the params
0x0804840c <main+24>: call   0x804830c <gets@plt>     //our call to gets
0x08048411 <main+29>: mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>: test   eax,eax
0x08048417 <main+35>: je     0x8048427 <main+51>
0x08048419 <main+37>: mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>: call   0x804832c <puts@plt>
0x08048425 <main+49>: jmp    0x8048433 <main+63>
0x08048427 <main+51>: mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>: call   0x804832c <puts@plt>
0x08048433 <main+63>: leave
0x08048434 <main+64>: ret

```

Hmmm, we better
[review](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html) some intel
syntax.
 - esp is the register for the stack pointer (32-bit x86).
 - the format is `instruction dst, src`
 - e.g., `mov DWORD PTR [esp+0x5c], 0x0` means move the 32-bit representation
   of 0x0 (DWORD) into the address `esp+0x5c`. 
 - `lea`: loads the effective address into the register rather than the content
   at that address.

We can surmise that location of the modified variable is `esp + 0x5c` because
of the initialization `mov` at 0x080483fd and `mov,test,je` sequences ending at
0x08048417. Similarly, we guess that `esp + 0x1c` is the start of `buffer`
because that address is passed to gets via the eax register getting pushed on
the stack, The 32-bit x86 **calling convention** demands that arguments are
passed to functions via the stack.

We know know the relative positions of `modified` and `buffer` in memory. Some
quick subtraction ( 0x5c - 0x1c ) tells us that the start of buffer and
start of modified are only 64 bytes apart. Thus, we know they are directly
adjacent. To avoid overwriting the old instruction pointer, we simply need to
modify our malicious input such that it only overflows into `modified` and not
any further:

```bash
python -c "print 'a'*65;" | /opt/protostar/bin/stack0
```

Let's use gdb to visualize what is going on. 

```bash
$ python -c "print 'a'*65" > 65.txt
$ python -c "print 'a'*64" > 64.txt

$ gdb /opt/protostar/bin/stack0
disass main
break *0x08048411 // right after call to gets
r < 64.txt
x/s $esp+0x1c // to see our string
x/68bx $esp+0x1c // to see the bytes in hex
//Warning, everything is flipped! higher addresses are lower
//The 0x00000000 right after the 'AAAA' is modified
c
r < 65.txt
x/68bx $esp+0x1c
c
```

### Setuid Binaries

Why is it that we have don't have permission to read "flag.txt" but when
exploit the binary when read "flag.txt"? In other words, why does a binary
(that we can run) have different permissions than we do?

First, let's take a look at the stack4 binary using the `file` utility: `file
./stack4`. The output includes a curious bit of text calling stack4 a "setuid"
ELF executable.  What does "setuid" mean here? To find out, we start the same
way we always do on Linux: reading the MAN page. 

The man page gives us more information (specifically, about the C library
funciton, but they are related). We can see that setuid stands for set user
identity. It allows a process to run as if it was started by a particular user
and, consequently, run with that user's permissions.  Setuid binaries are
useful for acheiving **priviledge escalation**. 

If we run `id` we can see information about the current user, including their
user id and group id. We can use `cat /etc/passwd` to see all of the users on
the system. For example, we see that "root" has a userid of 0. 

Fortunately, a malicious user cannot simply write a program and use setuid to
make that program run as root. The OS has permissions in place to prevent that
(as we can posit from looking at the errors section of the man page). So what
is it good for? It is often used to allow an unpriviledged user to access
hardware features or to temporarily give a user elevated priviledges, e.g.,
`ping` and `sudo` are both setuid binaries.  For an attacker, though, if he can
exploit a setuid binary that is running as root, then he effectively has root
priviledges. 



