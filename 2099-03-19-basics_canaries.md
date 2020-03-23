---
title:  "Lecture Notes: Basics of Stack Canaries"
date:   2019-03-19 09:00:00
categories: notes lecture
layout: post
---

How can we protect our binaries from the types of  attacks we have seen so far?
Assuming we can't prevent programmers from making the mistakes that lead to
buffer overflows, the most obvious approach is to make the bug harder to
exploit.

One possible way to use something called a **stack canary**. This is a special
value that the compiler places on the stack between the local buffers and the
saved return address. If an attacker overflows the buffer, the idea is that the
canary value will be overwritten. The program will check the canary value at
the end of function execution. If that value has been modified, the program
knows that something is wrong.   

Canaries, in general, have a number of appealing qualities. First, the machine
instructions for inserting and checking canary values can be inserted
automatically by the compiler. In other words, the programmer does not have to
modify her C code to use this defense (might need to set a compiler flag
though). Second, canaries can be implemented efficiently. We will talk about
that a bit more in just a bit.

Canaries are no panacea, however. They are very limited in the types of attacks
they can stop. For instance, canaries on the stack won't protect against other
types of memory errors (e.g., a write-what-where style of bug) or overflows on
the heap. Further, they won't protect against information leakage, e.g., buffer
overflows that involve reading memory past the end of a buffer rather than
writing information.  Further, they can be useless if the attacker can guess
the value of the canary.

### Picking a Canary Value 

One interesting design question: what value should the canary have? We usually
don't want to the attacker to be able to guess the value; otherwise, the
attacker could simply overflow the buffer and overwrite the canary in memory
with the exact same value. 

One way to address the guessing issue to use a random value that is determined
at the start of the program. This value is then used for all function calls.
These canaries are hard to guess, but vulnerable to **information leakage**.
Of course, you also have to find a way to protect the canary value in memory.
You can make the canary more robust by using a different random value for every
function call, but this adds (a bunch) of additional overhead and if you had a
magically-fast and protected region of memory you'd probably just directly save
the return address rather than the canary (and call it a shadow stack ;).

Another way to address the guessing issue is to make a
**terminator canary** that is composed of characters like `CR, LF, null bytes`
etc. The idea behind a terminator canary is to stymie the overflow step because
`scanf` and other vulnerable functions will stop reading input on these
characters and thus the attacker can't use them as part of his malicious input.


### Implementation of Canaries

When talking about defenses, the amount of overhead a software defense adds can
determine how likely that defense is to be used in practice. A good rule of
thumb is that the defense must add less than 10% overhead to be adopted. Here
we are typically more concerned about a defense's  execution overhead and not
its storage overhead. 

So why are canaries cheap? I.e., why do they have low execution overhead? To
understand, we really need to focus on a specific implementation and look at
 - how many instructions that implementation adds (broadly)
 - how many of those instructions result in a memory access.  While we are
   oversimplifying the analysis, in general, memory accesses are incredibly
slow.

```
# In Proluge: grab the canary value, using the segment register to find it
mov rax, fs:28h
mov [rsp+38h], rax

# In Epiloge: grab the correct canary, compare it to the canary on the stack
mov rax, [rsp+38h] 
xor rax, fs:28h
jz ...
call ___stack_chk_fail

```

Here we use the segment register (in part) because it allows us to set the
canary at runtime without changing the code section.




