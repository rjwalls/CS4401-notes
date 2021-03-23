---
title:  "Lecture Notes: Basics of Stack Canaries"
date:   2020-01-06 01:01:01
categories: notes lecture
layout: post
challenges: not-so-random ssp-buffer
---

How can we protect our binaries from stack-based buffer overflows?  Assuming we
can't prevent programmers from making the mistakes that lead to buffer
overflows, the most obvious approach is to make the bug harder to exploit.

One possible way to use something called a **stack canary**. This is a special
value that the compiler places on the stack between the local buffers and the
saved return address. If an attacker overflows the buffer, the idea is that the
canary value will be overwritten before the attacker reaches the return
address.  The program will check the canary value at the end of function
execution (in the epilogue). If the stored value is different from what the
program expects, the program knows that something is wrong.   

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
writing information.  Further, they can be bypassed if the attacker can guess
the value of the canary. Finally, they don't actually stop the overflow from
occurring, they just detect it after the fact.

### Picking a Canary Value 

One interesting design question quickly arises: what value should the canary
have? We don't want to the attacker to be able to guess the value; otherwise,
the attacker could simply overflow the buffer and overwrite the canary in
memory with the exact same value to avoid detection. 

One way to eliminate the possibility of the attacker guessing the canary's
value is to use a random value that is determined at the start of the program.
All the program needs to do it save the correct canary value somewhere in
memory, let's call this the *canonical canary*, and then place a copy of that
value on the stack for every function call.  These canaries are hard to guess,
but are still vulnerable to **information leakage**.  Of course, you also have
to find a way to protect the canonical canary value in memory---imagine what
would happen if the attack could change the canonical canary!  

You can make canaries more robust by using a different random value for every
function call, but this adds (a bunch) of additional overhead (i.e., the
program takes longer to execute)  and if you had a magically-fast and protected
region of memory for the canonical canary you'd probably just directly save the
return address rather than the canary (and call it a **shadow stack**).

Another way to address the guessing issue is to make a **terminator canary**
that is a canary composed of characters like `CR, LF, null bytes` etc. The idea
behind a terminator canary is to stymie the overflow step because `scanf` and
other vulnerable functions will stop reading input on these characters and thus
the attacker can't use them as part of his malicious input. Such canaries are
bit more robust against memory leakage, but obviously, this technique won't
help with `gets()` or write-what-where vulnerabilities. 


### Implementation of Canaries

When talking about defenses, the amount of overhead a software defense adds can
determine how likely that defense is to be used in practice. A good rule of
thumb is that the defense must add less than 5% overhead to be adopted. Here we
are typically more concerned about a defense's  execution overhead and not its
storage overhead. 

So why are canaries cheap? I.e., why do they have low execution overhead? To
understand, we really need to focus on a specific implementation and look at
 - how many instructions that implementation adds (broadly), and
 - how many of those instructions result in a memory access.  

While we are oversimplifying the analysis, in general, memory accesses are
incredibly slow and thus worth minimizing.

Here is GCC's implementation of the stack canary as seen in a simple function:

```
# In the function's Prologue: 
# grab the canary value, using the segment register to find it
mov rax, fs:28h
mov [rsp+38h], rax

# In the function's Epilogue: 
# Grab the correct canary, compare it to the canary on the stack
mov rax, [rsp+38h] 
xor rax, fs:28h
jz ...
call ___stack_chk_fail

```

This implementation uses  the segment register `fs`, in part, because it allows
the canary to be set at runtime without needing to recompile the code. 

Note, the GCC canary implementation only requires two additional instructions
to be added to the prologue of each protected function and a smattering more to
be added to each epilogue. Further, the prologue only requires a single read
and write to memory while the epilogue requires two additional reads and no
additional writes to memory. In short, the performance impact is minimal. 
