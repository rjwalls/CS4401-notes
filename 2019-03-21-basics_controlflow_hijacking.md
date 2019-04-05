---
title:  "Lecture Notes: Basics of Control-flow Hijacking"
date:   2019-03-21 09:00:00
categories: notes lecture
layout: post
---

So far we've talked (a lot) about one specific bug related to memory
management: buffer overflows. We've discussed how a simple buffer overflow can
be used by an attacker to take control of a program's execution. In other
words, we introduced the most basic example of a **control-flow hijacking**
attack.

Today we are going to talk more about memory management in C
and how memory mis-management can lead to many different types of errors. 

Memory is allocated in multiple ways in C-based languages:
 - Automatic, e.g. local variables in a function
 - Static, e.g., global variables
 - Dynamic, .e.g, malloc

The programmer is responsible for using allocated memory appropriately,
including checking the bounds of memory accesses and types. The programmer is
also responsible for the correct allocation and deallocation of dynamic memory.
In contrast, the compiler handles the allocation and deallocation of local
variables via instructions in the prologue and epilogue.

Unfortunately, memory management is error prone and different types of bugs can
occur:
 - buffer overflows: writing past the bounds of the allocated memory
 - dangling pointers: pointers to deallocated memory (that the program will
   then use as if it is valid memory)
 - double frees: deallocating memory twice
 - memory leaks: never deallocating memory 

With the (probable) exception of memory leaks, an attacker can exploit each
type of bug to take control of a program.

**Aside:** This lecture is based, in part, on Yves Younan's [talk on C
exploits](https://www.youtube.com/watch?v=tVG8e_Dneag). Consider taking a look
at LiveOverflow's [description of the GOT and
PLT](https://www.youtube.com/watch?v=kUk5pw4w0h4), and Izik's [treatise on
abusing CTORS and DTORS](https://www.exploit-db.com/papers/13234/).

### Code Pointers

So far our attacks have primarily focused on overwriting the saved return
address on the stack to hi-jack control flow. This saved return address is an
example of a **code pointer**. We can broadly define a code pointer as a
program variable that stores a code address and that address is intended to be
loaded into the instruction pointer.  

Some examples of code pointers. 
 - Return addresses: the saved address of where execution must resume when a
   function ends.
 - Function pointers: the C variable used to dynamically specify which function
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

The moral of the story is that there are many memory locations that an attacker
can target to hijack control flow. 


### Indirect Pointer Overwriting

Let's talk about how an attacker can leverage *data pointer* to create a
write-what-where vulnerability and bypass stack canaries.

In the following example, we are going to manipulate a data pointer to allow us
to write to any value to any memory location. Let us assume that there is a
stack canary in place to prevent us from overwriting the return address. 

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
we have all the capabilities we need to bypass the canary.

Specifically, we need to overwrite `buffer` to make `b` point to the saved
return value on the stack. The program will then overwrite the saved address
with our command line argument---which should be the address of win(). At the
end of the main function, the stack should look like the following:

```
+--------------------------------------+-------------------+
|buffer                            | b |              |ret |
+------------------------------------+-+-------------------+
                                     |                ^
                                     |                |
                                     +----------------+

```



### Heap-based buffer overflows

The previous example of indirect pointer overwriting probably felt a little
contrived. However, that type of vulnerability shows up quite naturally when
using the heap. To understand why, we need to understand how the heap works. 

The heap contains dynamically allocated memory. This memory must be explicitly
managed by the programmer, e.g., using calls to malloc() and free(). For
example, the following code will allocate some memory on the heap and return a
pointer to the calling function.

```c
int * array = malloc(10 * sizeof(int));
```

When the program no longer needs that allocated memory, it can be freed using:

```c
free(array);
```

Notice that free simply takes a pointer to the memory. How does free know how
big the allocated memory chunk was?  Importantly, each chunk of allocated
memory is associated with some metadata that describes information about that
chunk, e.g., the size. Often that metadata is stored in a header that is
adjacent to the chunk in memory. Given a pointer to allocated memory, `free()`
just needs to look at a fixed offset from the pointer to access the metadata
and get the size of the chunk. Further, if an attacker overflows a heap buffer,
then they can overwrite that metadata. To better visualize how the heap works, 
watch this [LiveOverflow video][liveoverflow-heap]. 

[liveoverflow-heap]: https://youtu.be/HPDBOhiKaD8

But wait...when we overflow on the stack, we directly manipulate the saved
return address to hi-jack the control flow...but there are not any return
values on the heap. So how can heap-based buffer overflows help us if there
aren't any return values? We are going to construct a write-what-where
vulnerability that takes advantage of this behavior.

`malloc` is going to start its operation by grabbing a large, contiguous,
region of memory of from the OS to form the heap. Whenever a function calls
`malloc`, `malloc` is going to take a piece of that memory, called a **chunk**,
and return a pointer to the calling function. You can visualize the memory like
this:

```
+----------------------------------------------------------------------------+
|XXXXX       XXXXXXXXXX     XXXXXXXXXXXXXXXXX        XXXXXXXXXXXX            |
+----------------------------------------------------------------------------+
         XX: Used Chunk

```

Notice that free and used chunks are interspersed throughout the heap. If we
take a closer look at a single chunk (one that is in use) we can see what the
header looks like.


```
CHUNK 1: In use

               pointer returned
               by malloc()
               ^
size of prev.  |
chunk          |
    ^          |
    |          |
    +----+----++----------------------------------------------------------+
    |    |    | USER DATA....                                             |
    +----+-+--+-----------------------------------------------------------+
           v
           size of this
           chunk

```

If we free chunk 1, the first part of the user data gets overwritten with a
couple of pointers. These pointers are used to insert the chunk into
doubly-linked list of free chunks, often called a **freelist**. `malloc` will
use this list of free chunks to allocate memory when `malloc()` is called. 


```
CHUNK 1: After being freed

              Prev chunk in freelist

                 ^
                 |
                 |
+----------------+----------------------------------------------------------+
|sizp|size| FWD| BCK| Old user data...                                      |
+----------+----------------------------------------------------------------+
           |
           |
           v

        Next chunk in freelist

```

Note: the next chunk in the freelist is not necessarily adjacent in memory to
the current chunk. When we unlink a chunk from the freelist, we have to update
the pointers to remove the unlinked chunk. The key point is that `malloc` uses
the header information stored in the chunk to figure out what needs updating. 

Imagine that chunk 0 and chunk 1 are adjacent in memory. Further, chunk 0 is in
use while chunk 1 is free. An attacker can overflow a buffer in chunk 0 to
overwrite the forward and backward freelist pointers in chunk 1. If the forward
pointer is changed to point to the target return address (on the stack) and the
backward pointer is changed to point to the injected code (on the heap), the
freelist unlinking process will overwrite the return address with the address
of the injected code. Of course, the attacker needs to trigger the unlinking
process (i.e., force malloc to reallocate that memory).  


```
CHUNK 0                              CHUNK 1
  in use                               in free list
     +----------------------------------------+
     |                                        |
     |                             +          |
     V                             |          |
+---------------------------------------------+------------------+
|S|S|INJECTED CODE                 ||S|S|FWD|BCK|                |
+-----------------------------------------+----------------------+
                                   |      |
                                   +      |
                                          V
                                      +-------+
                                      |ret    |
                                      +-------+
```

