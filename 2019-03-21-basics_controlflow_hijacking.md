---
title:  "Lecture Notes: Basics of Control-flow Hijacking"
date:   2019-03-21 09:00:00
categories: notes lecture
layout: post
challenges: heap1r heap3r
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



### Indirect Pointer Overwriting on the Heap

The previous example of indirect pointer overwriting probably felt a little
contrived. However, that type of vulnerability shows up quite naturally when
using the heap. 

But wait...when we overflow on the stack, we directly manipulate the saved
return address to hi-jack the control flow...but there are not any return
values on the heap. So how can heap-based buffer overflows help us if there
aren't any return values?  The answer here is that we need to construct a write-what-where
vulnerability that will allow us to write whatever value we want to whatever
location we want. This would allow us to target a return address[^1] on the stack
and change the value to any address we choose.

[^1] Note, we don't have to target a return address on the stack. There are many other code pointer that might make better targets, e.g., an entry in the global offset table. We will talk about the global offset table in another lecture.

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

The buffer overflow in this code leads to write-what-where style vulnerability that is very similar to what we saw in the previous example. In particular, we can manipulate both where the `name` field from struct `i2` points as well as the value that gets written to that location. To understand how this attack is possible, we need to first understand how the heap works. For that, I invite you to watch this [LiveOverflow video on the basics of malloc][liveoverflow-heap]. 

[liveoverflow-heap]: https://youtu.be/HPDBOhiKaD8 



### Exploiting Malloc Unlinking

Another way to create a write-what-where vulnerability is to exploit the unlinking behavior of malloc's freelist.  
The basic idea of the attack is as follows. We are going to manipuate the metadata of the free memory chunks stored in the freelist such that when malloc re-allocates a particular chunk (i.e., unlinking that chunk from the freelist) malloc will write an attacker-provided value to an attacker-provided location. We give the full details below.

Recall that `malloc` is going to start its operation by grabbing a large, contiguous,
region of memory of from the OS to form the heap. Whenever a function calls
`malloc`, `malloc` is going to take a piece of that memory, called a **chunk**,
and return a pointer to the calling function. Also recall, that malloc write a header for each chunk so that it can keep track of important metadata.

If we take a closer look at a single chunk (one that is in use) the
header looks like the following:


```
CHUNK FOO: In use

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
           
Note: In some version of malloc, the lowest order bit is set to indicate if the previous chunk is in use.

```


Whenever the program makes a request via `malloc()`, malloc is going to pick one of the free chunks and give it to the process. When the program `frees` memory, malloc adds it back to the list of free chunks.
After the process has been executing for a while---and a number of chunks have been allocated and freed---the heap will start to look  something like this:

```
+----------------------------------------------------------------------------+
|XXXXX       XXXXXXXXXX     XXXXXXXXXXXXXXXXX        XXXXXXXXXXXX            |
+----------------------------------------------------------------------------+
         XX: Used Chunk

```

Notice that free and used chunks are interspersed throughout the heap, so malloc has to use some mechanism to keep track of where the free chunks are in memory. This mechanism is a doubly-linked list of free chunks called a **free-list**.

Now imagine a scenario where the program has freed a particular chunk. Malloc will update the header of the now-freed chunk to add forward and backward pointers, inserting the chunk into the freelist. These new pointers actually overwrite the first 8 bytes of the user data that was stored in the chunk (before it was freed). 

```
CHUNK FOO: After being freed

               Next chunk in freelist
               ^
size of prev.  |
chunk          |
    ^          |
    |          |
    +----+----++----------------------------------------------------------+
    |    |    | FWD | BCK |   Old data                                    |
    +----+-+--+-----------------------------------------------------------+
                     |
                     v
                     Previous chunk in freelist

```

Keep in mind that free chunks are likely scattered throughout the heap, so the next chunk in the freelist (i.e., the chunk pointed to by`FWD`) is not necessarily adjacent in memory to `chunk foo`. 

When we unlink a chunk from the freelist, we have to update
the pointers to remove the unlinked chunk. The key point is that `malloc` uses
the `FWD` and `BCK` pointers stored in the chunk  to figure out what needs updating. 

Imagine that chunk bar and chunk foo are adjacent in memory. Further, chunk bar is in
use while chunk foo is free. An attacker can overflow a buffer in chunk bar to
overwrite the forward and backward freelist pointers in chunk foo. If the forward
pointer is changed to point to the target return address (on the stack) and the
backward pointer is changed to point to the injected code (on the heap), the
freelist unlinking process will overwrite the return address with the address
of the injected code. Of course, the attacker needs to trigger the unlinking
process (i.e., force malloc to reallocate that memory).  


```
CHUNK Bar                              CHUNK Foo
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


**A major caveat:** Real malloc implementations are a lot more complicated than what we have presented here. The specific behavior we  want to exploit will depend on the implementation of malloc (and there are a lot of implementations). For example, this LiveOverflow video describes how [dlmalloc handles fastbins][fastbins]. 

[fastbins]: https://youtu.be/gL45bjQvZSU
