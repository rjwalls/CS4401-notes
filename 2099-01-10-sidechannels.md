---
title: "Lecture Notes: Side-Channel Attacks"
date: 2020-01-07 01:00:00
categories: notes lecture
layout: post
author: Edward Krawczyk
challenges: stopwatch
---

We are now going to explore a different class of exploits, called **side-channel
attacks**. Like before, we try to leverage a flaw in the implementation of a
program. Unlike the previous attacks that have been covered, we often do not
seek to redirect program execution. Now, we usually have a different goal: we
only aim to recover information that we are not supposed to have access to.
While this sounds pretty useless (what's the point of an attack if you can't
pop a shell or get root access?) it can lead to catastrophic data breaches. For
example, your web traffic (when using HTTPS) is encrypted with a secret key.
This prevents any third party from snooping on you. If this secret key is
leaked, though, anybody can decrypt your web traffic.

Side-channel attacks do not use any form of binary exploitation to accomplish
this. These attacks do not directly try and read out information from a
program (like we might do with a format string vulnerability). Instead they try
and obtain information _related_ to the data they want [1,2]. This is an odd
concept, so we begin with an example.

The classic analogy is a poker face. When you play poker, you do not know the
cards your competitor has in their hand. You want to know what cards they have,
though, so you can determine if your hand is better than theirs. You obviously
cannot look at their hand, so what can you do? Well, perhaps you can examine
their facial expression, or body language. If they have a good hand, they might
appear confident, while a bad hand might cause them to look a little nervous.
Of course, they might also have a good poker face, but we're going to ignore
that.

Reading an opponent's face is a side-channel attack. Instead of directly
determining what cards they have, you determine information _about_ those
cards: whether they form a good or bad hand. We call the medium used to obtain
this information a side-channel. Here, the face is a side-channel. Formally, a
side-channel leakage is an unintended source of information _about_ some
secret. This secret may be a password, encryption key, or PIN. The medium that
the information is leaked through is the side-channel

Now, we will examine side-channel attacks on software, in the case where the
side-channel is time. There are other side-channels that can be exploited to
attack software, notably caches.

### An example attack

An _timing_ side-channel attack assumes that the time that is consumed to
perform a task leaks information about the data that task is being performed on
[1,2]. Consider a simple bit of code that checks if a password is correct:

```
int password_checker (char* input_pass, char* saved_pass) {
	for (int i = 0; i < PASS_LEN; i++) {
		if (input_pass[i] != saved_pass[i])
			return BAD_PASS; // short circuit for efficiency!
	}
	return GOOD_PASS;
}
```

Here, we compare the two passwords character by character. If two characters do
not match, we return early and don't bother comparing the rest of the
passwords. Note that this is very similar to how the standard `strcmp()`
function works in C. While this saves time and is efficient, it leaks
information about the password. If we enter a password where only the first
few characters are correct, `password_checker` will take a little bit longer to
return a value than if we enter a password where the first character is
incorrect. So, the time this code takes to return gives us information about
how many leading characters in an incorrect password are correct.

With this timing difference, we can put together an attack that can correctly
guess the password in a relatively small number of tries. To start, we try
entering all passwords with all possible leading characters. If the passwords
are alphanumeric, we would try all passwords from 'Abcd...', 'Bbcd...', ...
'0bcd...', 9bcd...'. We keep the remainder of the password constant to isolate
the effect of just this first character. Timing how long it takes to evaluate
each password, we will notice that one of these takes slightly longer than the
others. That password guess is the one with the correct first character. To
understand why, notice that if the first character is correct, the function
moves on and compares the second characters. Since an incorrect guess will
terminate the function after just examining the first character, a password
with the correct first character (but an incorrect second character) will take
roughly twice as long to evaluate.

Now that the first password character is determined, we target the second
character. Like before, we try all passwords with all possible characters in
the second position. However, for all of these guesses, we fix the first
character to be what we previously determined to be the correct first
character. Again, we will notice that one guess takes a little longer to be
processed, indicating that it has the correct second character. We then repeat
this process for the remaining characters in the password.

**Wait a minute, isn't this just brute forcing the password?** Yes, and no. It
is intelligently brute forcing the password over a much much smaller search
space than a simple brute force attack. For a basic brute force search, we
would evaluate every possible password. For a 16 character alphanumeric
password, this would mean testing up to 36^16 = 7.96 _ 10^24 possible
passwords. With our approach, we brute force each character individually,
cutting down the maximum number of tries to 36 _ 16 = 576. That is a lot
smaller, and we will take seconds, not years, to run.

### Constant-time code

The password checker example given above may seem a little contrived, and any
good password checker won't behave like this. To provide another real-world
example of a vulnerability, we look at a problem in GnuPG 1.4.13's
implementation of the RSA algorithm. RSA is a cryptographic algorithm that
requires performing exponentiation on a modular space. An efficient algorithm
to perform this operation is the square and multiply algorithm, shown below in
a very simplified manner. We can see that an additional two lines of code is
executed, depending on the value of a secret. This secret is actually a bit of
the private encryption key [3].

```
long exponent(int b, int e, int m) {
	long x = 1;
	for (long j = e - 1; j >= 0; j--) {
		x = x*x;
		x = x % m;
		...
		if (super_secret_bit_j == 1) {
			x = x*b;
			x = x % m;
		}
		...
	}
	return x;
}
```

It turns out that this cannot really be exploited by strictly analyzing
execution time, the attack is more complex and requires probing the processor's
cache. This attack, called Flush+Reload, will be mentioned in the next section.
Regardless of how the attack is carried out, the code is still made vulnerable
by non-constant time execution, that _depends on a secret value_. For a
discussion on writing code that securely operates on secret data, read [2].

### Hardware Side-Channels

Flush+Reload is a great introduction into the world of _hardware_ side-channel
attacks. These are attacks that use information gained from physical system
hardware, such as processor cache. Flush+Reload and other so-called cache
attacks time how long it takes to access a piece of data, to determine whether
that data is in the processor's cache, or system memory. Recall that data held
in cache will be accessed **much** faster than anything in memory. So, if it
takes a very short time to access some data, we can infer that data is held in
cache. Alternately, a long access time means that data was not present in
cache, so we had a cache miss and had to load the data in from the slow memory
or disk.

Flush+Reload depends on two important system features:

1. Some level of processor cache is shared between programs running on a
   system. This is always true for current intel processor designs.

2. The OS has page de-duplication enabled. This means that redundant pages in
   memory are consolidated. So, if two programs are both using some function
   from libc, page de-duplication will mean both programs access the same memory
   page that contains this code. The idea here is that this cuts down on system
   memory usage, as otherwise both programs would each have an independent copy of
   the same memory page. Note that this feature is now often disabled by default.

You may see where we are going with this. In order to determine whether a
victim process is using a piece of information, we time how long it takes for
us to access that information. In this context, the information is a memory
page with a portion of code from GnuPG. In order to more accurately time this
access, the attacker process first uses the `clflush` instruction to clear the
memory line of interest from cache. Then, we try and access this memory line.
If the victim process has used this memory line since we cleared it from cache,
we will have a very fast access time. This is because the victim's access will
have placed the memory line back in cache. If the victim did not access the
memory line, we will have to load it in from memory, and will observe a slow
access time. Coupling this attack with a secret-dependent code condition (see
the `exponent()` function), we are able recover secret keys. To see how a full
key recovery is performed, see the paper, [3].

It is important to note that while this attack is mitigated by disabling page
de-duplication, many other hardware attacks are ever-present. This is because
hardware cannot be patched to accommodate a security concern. Remember Meltdown
and Spectre? [6]

## Using Symbolic Execution for Binary Analysis

Beyond side-channel attacks, another powerful technique for understanding and exploiting binaries is **symbolic execution**. Symbolic execution allows us to analyze a program's behavior without providing concrete input values, instead treating inputs as symbols and computing constraints that determine program behavior.

### What is Symbolic Execution?

Symbolic execution simulates program execution using symbolic values (variables) instead of concrete values. As the program executes, the symbolic executor builds up constraints based on the program's execution path. For example, if a program contains:

```c
if (x > 10) {
    // Path A
} else {
    // Path B
}
```

The symbolic executor will explore both paths and record that Path A requires the constraint "x > 10" while Path B requires "x ≤ 10".

### Angr: A Python Framework for Symbolic Execution

Angr is a powerful Python framework that provides tools for binary analysis, including symbolic execution. It can help with tasks such as:

1. Finding paths through a binary that reach specific addresses
2. Determining input values needed to reach certain code locations
3. Identifying exploitable vulnerabilities
4. Bypassing complex authentication checks

### Example: Finding Valid Password Inputs

Consider a simple program that checks if a password is correct:

```python
import angr

# Create an Angr project
proj = angr.Project('./password_check', auto_load_libs=False)

# Find the addresses we want to reach and avoid
main_addr = proj.loader.find_symbol('main').rebased_addr
target_addr = 0x4006ed  # Address of success message
avoid_addr = 0x400790   # Address of failure message

# Create a symbolic bitvector for our input
password_len = 16
password = proj.factory.claripy.BVS('password', password_len * 8)

# Create an initial state starting at main
state = proj.factory.entry_state(args=['./password_check', password])

# Create a simulation manager
simgr = proj.factory.simulation_manager(state)

# Explore until we find our target or hit a dead end
simgr.explore(find=target_addr, avoid=avoid_addr)

# If we found a solution, get the input value
if simgr.found:
    solution_state = simgr.found[0]
    solution = solution_state.solver.eval(password, cast_to=bytes)
    print(f"Found password: {solution.decode()}")
else:
    print("No solution found")
```

This code tells Angr to find a path through the binary that reaches the success message while avoiding the failure message, then solves for the input value (password) that would take that path.

### Limitations of Symbolic Execution

While powerful, symbolic execution has limitations:

1. **Path explosion**: As program complexity increases, the number of possible paths grows exponentially
2. **Environment modeling**: Symbolic execution must model the operating system, libraries, and other aspects of the runtime environment
3. **Complex constraints**: Some program behaviors create constraints that are difficult for SMT solvers to handle
4. **Time and memory requirements**: Exploring all paths in a non-trivial program can be resource-intensive

Despite these limitations, symbolic execution is a valuable tool for binary analysis and exploitation, especially when manual analysis becomes too complex or time-consuming.

### Summary

Side-channel attacks represent a powerful class of exploits that don't rely on direct memory corruption but instead extract information through observable side effects of program execution:

1. **Timing Side Channels**

   - Extract information based on execution time differences
   - Can reveal password comparisons, cryptographic keys, or control-flow decisions
   - Often exploited in password checkers and cryptographic implementations

2. **Cache Side Channels**

   - Techniques like Flush+Reload that measure memory access times
   - Can determine which memory locations are being accessed by a process
   - Have been used to extract cryptographic keys and break ASLR

3. **Hardware Side Channels**

   - Take advantage of physical properties of computing systems
   - Include power analysis, electromagnetic emissions, and acoustic analysis
   - Often effective against embedded systems and security hardware

4. **Symbolic Execution**
   - Not a side channel, but a powerful analysis technique
   - Treats inputs as symbols rather than concrete values
   - Explores multiple program paths simultaneously
   - Can automatically find inputs that trigger specific behavior
   - Tools like Angr can automate the process of finding exploitable paths

Side-channel attacks demonstrate that security vulnerabilities can exist even in programs without memory corruption bugs. Modern defense strategies must consider side-channel resistance alongside traditional security measures. While symbolic execution isn't itself a side channel, it provides powerful capabilities for analyzing binaries and finding paths to exploitation when manual analysis becomes too complex.

### References and Further Reading

[1] [A discussion on constant time code](https://www.bearssl.org/constanttime.html)

[2] [Constant time code, and what to consider when writing it](https://www.chosenplaintext.ca/articles/beginners-guide-constant-time-cryptography.html)

[3] [Flush+Reload, and attacking RSA with it](eprint.iacr.org/2013/448.pdf)

[4] [A very brief into to a different type of side-channel, power consumption of embedded devices](https://www.rambus.com/blogs/an-introduction-to-side-channel-attacks/)

[5] [Side-channel attacks developed in the ECE dept. at WPI](http://vernam.wpi.edu/publications/)

[6] [Meltdown and Spectre](https://meltdownattack.com/)
