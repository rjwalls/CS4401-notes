---
title: "Lecture Notes: Basics of Format String Vulnerabilities"
date: 2019-03-22 09:00:00
categories: lecture notes
layout: post
author: Juan Luis Herrero Estrada
---

Buffer overflows are not the only programming error that an attacker can leverage to exploit binaries. In these notes, we introduce the concept of **string format vulnerabilities** and describe how they can be used to both leak information from memory and modify *arbitrary* locations with *arbitrary* values. In short, string format vulnerabilities offer opportunities beyond what is possible for a simple buffer overflow. 

One of the most common functions used in c is `printf`. In introductory Systems
courses we learn how to read input from a file or stdin, perform some string
manipulation, and display the result on the command line using `printf`. Most
of the time you've used format characters such `%s` for strings, `%d` for
integers, `%f` for floats. As well, the number of variables to print matches
one-to-one with the number of format characters in the string. When there
are more format characters than variables you can accidently (or purposely)
read and write into memory. More specifically, on the stack. Variables in
a C program are placed on the stack. `printf` and other variations of this
function (see `man 3 printf` for more information) read said variables from the
stack and substitutes them for the respective format character. Consider,

```c
char buffer[64];

buffer = "%s %s";

printf(buffer);
```

The function `printf` will read from the stack and interpret the bytes as
a string (an array of char with a null terminator at the end). Most likely,
this will cause a segmentation fault. Now, if instead we used the format
character `%x`, which prints out hexadecimal values (addresses) we can get
something more useful. We could read the addresses and values (in hexadecimal)
that are placed on the stack. We can gain sensitive information from a program
that wasn't previously available.

`printf` has another interesting format character called `%n` which writes the
number of printed characters before `%n` is seen to an address. Which address 
are we writing to? For example, consider the code

```c
int value;
printf("what's my age again? %n \n", &value);
printf(value = %d\n", value)
```

and the output of this would be

```c
what's my age again?  
value = 21
```

In the previous example we provided the address of the value variable but what
if none is given explicitly? Well, `printf` will try to use the next address on
the stack and write to it most likely ending in a segmentation fault. This
leads to 2 questions: where is user input placed in memory? and can we point
where `%n` writes to? The answer of the first question is it depends. The
C language does not define where in memory command line arguments are stored.
Although, information is normally stored on the stack. You can easily find this
out by using a sequence of characters that you can easily recognize as input to
the program you are testing. For instance you could use as input `AAAA %x %x %x
%x BBBB` since you'll quickly recongize that `0x41` is hex for A and `0x42` is
hex for B. If user input is stored on the stack then we can pass in the address
we want `%n` to write to. This could lead to overwriting a variable on the
program leading unauthorized access or we could even overwrite the global
offset table to jump to functions and code sections we weren't supposed to. 


### Example

Consider the following C code

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

We quickly can tell that the problem lies with `printf(string)` since we're
providing input from the command line. Lets try running the problem using
format characters we just learned about. So if we run 

`$ ./binary "%x %x %x %x"`

we get the result

`1 c2 f7ea48fb ffffd5ee`

which is interesting, but does not tell us if our input is on the stack. We probably
need to print out more of the stack. So we run the following

```./format1 "`python -c "print  'AAAA' + 'BBBB' +'%x ' * 200"`" | grep 41```

and we see that the output includes

`41410031 42424141 78254242 20782520 25207825 78252078 20782520`

that's definitely our input with a mask. Now we want to find the address of the
variable target so we can overwrite its result `%n` since we know this variable
lives on the stack. We get its address `0804a02c` by doing

`objdump -t binary | grep target`

Now lets add our address to our input string and we must not forget about
endianness. So we do 

```./binary "`python -c "print  'AAAAAAAA' + '\x2c\xa0\x04\x08' +'%x ' * 250"`"```

We can see a bunch of addresses and in the middle we get

`41414141 41414141 804a02c 25207825 78252078`

Now I will leave up to you find the right padding needed so when you add `%n`
to the end of you input string the address in question is `0x0804a02c`. This
will change the value of the `target` and thus execute code previously not
accesible to us.


### Defense

You want to sanitize user input before using it. The simplest approach in C to
avoid this type of attack is to use a format character as well! Instead of 

```c
printf(user_input)
```

we should do

```c
printf("%s", user_input)
```

If the user passed format characters as their input it will simply interpret read
it as a string (due to `%s` being used). No sensitive information is leaked and no
values on the stack are overwritten. As well, more modern compilers check for
unsafe uses of format strings.

There are existing defenses like stack canaries and ASLR to prevent a malicious user
from exploiting your program. Stack canaries could prevent somebody from using
`%n` to overwrite data on the stack. However, if you don't sanitize your user
input it could leak information about key objects in memories and thus help
an attacker overcome ASLR.

### Extra Examples

To see a more in depth walkthrough of the ideas from this article try following along with these writeups:

1. [format3](/writeup/format3_writeup.html)

### Sources

For more information about format strings and associated attacks, consider taking a look at the following links.

1. https://www.cs.virginia.edu/~ww6r/CS4630/lectures/Format\_String\_Attack.pdf
2. https://stackoverflow.com/questions/18681078/in-which-memory-segment-command-line-arguments-get-stored
3. https://liveoverflow.com/binary\_hacking/protostar/format4.html
4. https://liveoverflow.com/binary\_hacking/protostar/format1.html
