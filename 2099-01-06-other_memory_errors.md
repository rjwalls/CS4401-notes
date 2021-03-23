---
title:  "Lecture Notes: Other Memory Errors"
date:   2020-01-06 02:01:00
categories: notes lecture
layout: post
challenges: heap1r heap3r
---

So far we've talked (a lot) about one specific bug related to memory
management: buffer overflows. We've discussed how a simple buffer overflow can
be used by an attacker to take control of a program's execution. 

In these notes, we are going to talk more about memory management in C and how
memory mismanagement can lead to many different types of errors. 

Memory is allocated in three basic ways in C-based languages:
 - automatically, e.g. local variables in a function;
 - statically, e.g., global variables;
 - dynamically, .e.g, calls to `malloc`.

The programmer is responsible for using allocated memory appropriately,
including checking the bounds of memory accesses and types. The programmer is
also responsible for the correct allocation and deallocation of dynamic memory.
In contrast, the compiler handles the allocation and deallocation of local
variables via instructions in the function prologue and epilogue.

As with any process that relies on humans for correctness, memory management is
error prone. Buffer overflows are just one type of bug that can occur.  Others
include:
 - buffer overflows: writing past the bounds of allocated memory objects;
 - dangling pointers: pointers to deallocated memory (that the program will
   then use as if it is valid memory);
 - double frees: deallocating memory twice;
 - memory leaks: never deallocating memory;
 - uninitialized reads: reading memory before it has been initialized.

With the (probable) exception of memory leaks, each of these bugs may allow an
attack to take control of a program.

### Code Pointers

So far our attacks have primarily focused on overwriting the saved return
address on the stack to hi-jack control flow. This saved return address is an
example of a **code pointer**. We can broadly define a code pointer as a
program variable that stores a code address and that address is intended to be
loaded into the instruction pointer.  

Some examples of code pointers: 
 - Return addresses: the saved address of where execution must resume when a
   function ends.
 - Function pointers: C variables used to dynamically specify which function
   to execute.
 - Global offset table: addresses here are used to execute dynamically loaded
   functions (lazy loading: contains stub code that will load the function
into memory on the first call).
 - Virtual function table: addresses here are used to know which method to
   execute, e.g., dynamic binding in C++.
 - Destruction functions (i.e., `dtors`): these functions are called when a
   program executes.
 - Data pointers: these are not code pointers, per se, but they can be made to
   point to a code pointer and used to overwrite that pointer.

The takeaway here is that there are many memory locations that an attacker can
target to hijack control flow. 

### Indirect Pointer Overwriting

Let's talk about how an attacker can leverage a *data pointer* to create a
**write-what-where** vulnerability and bypass stack canaries. In other words,
in the following example, we are going to manipulate a data pointer to allow us
to write to any value to any memory location. Here we assume there is a stack
canary in place that prevents us from overwriting the return address with a
simple buffer overflow. 

```c

#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

static unsigned int a = 0xdeadbeef;

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  int *b = &a;
  char buffer[64];
  unsigned int ret;

  printf("buffer: %08x\n", &buffer);

  gets(buffer);

  if (argc > 1)
  {
    *b = strtoul(argv[1], 0, 16);
  }
}

```

In this code, there is a buffer overflow that allows us to write past the end
of `buffer`. We cannot use this bug to directly overwrite the saved return
value because of the stack canary.  However, we can overwrite the value of the
integer pointer `b` to point to any memory location we want. We can also
control the value that is written to `b` by supplying a command line argument,
e.g., `./vuln 0xfeedfeed`. Put these two capabilities together and we are able
to write an arbitrary value to an arbitrary location in memory. In other words,
we have all that we need to bypass the canary.

Specifically, we need to overwrite `buffer` to make `b` point to the saved
return value on the stack. The program will then overwrite the saved address
with our command line argument---which should be the address of `win()`. At the
end of the main function, the stack should look like the following:

```
+--------------------------------------+-------------------+
|buffer                            | b |              |ret |
+------------------------------------+-+-------------------+
                                     |                ^
                                     |                |
                                     +----------------+

```



### Indirect Pointer Overwriting on the Heap

The previous example of indirect pointer overwriting probably felt a little
contrived. However, that type of vulnerability shows up quite naturally when
using the heap. A write-what-where vulnerability on the heap would allow us to,
for example, target a return address on the stack and change the value to
any address we choose.

Note, we don't have to target a return address on the stack. There are many
other code pointers that might make better targets, e.g., an entry in the
global offset table. We will talk about the global offset table in another
lecture.

For example, consider the `heap1r` challenge:

```c
struct internet {
  int priority;
  char *name;
};

void winner()
{
  ...
}

int main(int argc, char **argv)
{
  struct internet *i1, *i2, *i3;

  i1 = malloc(sizeof(struct internet));
  i1->priority = 1;
  i1->name = malloc(8);

  i2 = malloc(sizeof(struct internet));
  i2->priority = 2;
  i2->name = malloc(8);

  strcpy(i1->name, argv[1]);
  strcpy(i2->name, argv[2]);

  printf("and that's a wrap folks!\n");
}


```

The buffer overflow in this code leads to write-what-where style vulnerability
that is very similar to what we saw in the previous example. In particular, we
can manipulate both where the `name` field from struct `i2` points as well as
the value that gets written to that location. To understand how this attack is
possible, we need to first understand how the heap works. For that, I invite
you to watch this [LiveOverflow video on the basics of
malloc][liveoverflow-heap]. 

[liveoverflow-heap]: https://youtu.be/HPDBOhiKaD8 

