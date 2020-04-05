---
title:  "Quick Notes: Help: My Exploit Works in GDB but not on the Server" 
date:   2020-04-01 01:01:00
categories: notes quick
layout: post
---

Stack addresses are going to be different in different execution environments.
You might find that your exploit string works in GDB, but segfaults outside of
GDB. Or maybe you got it working on your local machine but not on the server?
This is because the stack addresses are different (sometimes just a little bit
different) when you run in different environments. 

To understand why, remember that I said that environment variables are loaded
onto the stack when you launch a process. If different environments have
different values for those variables then the amount of stack space used to
store the environment variables will be different. Consequently, the stack
addresses further down in the stack will also differ. 

But many other factors can influence the state of the stack. Consider the
following test code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int i;

  printf("Stack is at %p\n", &i);
}
```

When compiled with `gcc test_stack.c -no-pie -o test_stack` and run in our
test environment with ASLR disabled, we see the following output:

```
$ ./test_stack
Stack is at 0x7fffffffe2c4
$ ./test_stack
Stack is at 0x7fffffffe2c4
$ /root/host-share/test_stack
Stack is at 0x7fffffffe294
$ ./test_stack POINTLESS_ARGUMENT
Stack is at 0x7fffffffe2b4
$ ENVVAR=Blah ./test_stack
Stack is at 0x7fffffffe2b4
$ ENVVAR=Blahddddddddddddddddddddddd ./test_stack
Stack is at 0x7fffffffe2a4
```

Notice how the address of `i` on the stack changes when we:
 - run with relative vs. absolute paths, or
 - add command line arguments, or 
 - add environment variables.

So what's the solution? It depends on the situation. If you are trying to get
the venerable stack-smashing-plus-shellcode attack to work, then you'll want to
employ a NOP sled---more details on NOP sleds in the classic paper "Smashing
the Stack for Fun and Profit". If you have a memory leak, then you can
(probably) directly figure out the location of the stack based on the contents
of the leaked memory. 

**Manipulating the environment variables.** It might also help to manipulate
the environment variables on the machine to make stack addresses more
predictable. You can:
 - view  environment variables with `printenv`, 
 - delete a variable with `unset env VARNAME`, 
 - change a value with `VARNAME='Foo'`
 - add new variable with `export VARNAME='foo'`
 - set a variable used for a single process `VARNAME='f' ./binary`.

In particular, GDB (at least some versions) sets two environment variables that
do not exist outside of GDB, `LINES` and `COLUMNS`. Sometimes, we can just
unset those two environment variables (when in GDB) and the memory addresses
will be the same when running the process inside of and outside of GDB---also
assuming you run the binary using the absolute path and same command line
arguments.


```
# In GDB
show environment
unset env LINES
unset env COLUMNS
