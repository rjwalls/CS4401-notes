---
title: "Machine Code Overview"
date: 2020-04-09 17:01:01
categories: notes lecture
layout: post
---

This is a Sparknotes version of machine code. It includes what registers are used for and what common instructions do in ATT syntax (sorry Intel fans!).
-   The program counter (%rip) tells the machine what to execute next
-   There are 16 total registers that store values for computation.
-   The condition codes store information about arithmetic operations and tell the conditional jumps what to do
-   Machine code is simply a series of bytes that we can parse as code because of how ELF binaries are set up
-   Different data types have different prefixes as well in ATT syntax

![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/data_types.png)

-   Operations access the Least significant bytes of the data they're mainpulating. For instance calling movb on a quad word would only take the least significant byte
-   Registers:
	-   These are all of the registers used by gcc
		-   rax, eax, ax, al: The return value of the function
		-   rsp, esp, sp, spl: The stack pointer
		-   Argument Registers
			-   r9(d,w,b): 6th argument
			-   r8(d,w,b): 5th argument
			-   rcx, ecx, cx, cl: 4th argument
			-   rdx, edx, dx, dl: 3rd argument
			-   rsi, esi, si, sil: 2nd argument
			-   rdi, edi, di, dil: 1st argument

	-   Callee saved: (The callee needs to have these values to run properly)

		-   rbx, ebx, bx, bl
		-   rbp, ebp, bp, bpl
		-   r12, r12d, r12w, r12b
		-   r13, r13d, r13w, r13b
		-   r14, r14d, r14w, r14b
		-   r15, r15d, r15w, r15b

	-   Caller saved:

		-   r10, r10d, r10w, r10b
		-   r11, r11d, r11w, r11b

-   Arithmetic address definitions:
	-   **$Imm:** The value of Imm (for instance 15 or 0xFF)
	-   **Ra**: the value of Register a
	-   **(Ra)**: the value in memory at the address in Ra. If Ra had 0xF, we would get the bytes starting at location 0xF in memory
	-   **Imm(Rb, Ri, S)**: The value in memory at address (Imm + Rb + Ri*s). If any part is missing, assume it is 0 or 1 (in the case of s)

-   Data Movement

	-   **mov(b,w,l,q)** S, D: Move S into D. D can be a register or memory. S can be an Immediate, Register, or Memory. These values can be arithmetic operations defined above as well. This only affects the bytes written by the source
	-   When moving a smaller amount of data into a larger destination, the extra bytes are left as 0.
	-   **movz(b,w)(w,l,q) S, D**: Moves S into D, with zeroes for leftover bytes
	-   **movs(b,w)(w,l,q) S, D**: Moves S into D, with the sign used for leftover bytes

-   The stack
	-   **pushq S**: equivalent to subq $8, rsp; move S, (rsp).  This moves the stack pointer down 8 bytes (remember the stack grows down), then puts S into the address stored in the stack pointer.
	-   **popq D**: equivalent to : (rsp), D; addq $8, rsp. This gets the quad word last put on the stack and puts it into D. It then moves the stack back up 8 bytes to free up the space

-   Arithmetic operations
	-   **leaq S, D** load address at S into D. S is an arithmetic address (see the above section) and D is a register.
	-   **SA_** Shift arithmetic (or logical) left or right
![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/arith_ops.png)
	- These are the other arithmetic operations. They are pretty self-explanatory from the diagram. In crazy cases you get 128 bit words which can be manipulated with these instructions
![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/full_arith_ops.png)

-   Conditional and flags

	-   In x86_64 we have multiple flags that can get set when we perform a comparison
	- ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/flags.png)

	-   **Flags** (consider the equation t=a+b for examples)
		-   **CF**: Carry flag: This gets set if the most recent operation generated a carry out (mostly used to detect overflow in unsigned operations) (t < a)
		-   **ZF**: the most recent operation yielded 0 (t==0)
		-   **SF**: The most recent operation yielded a negative value (t<0)
		-   **OF**: the most recent operation causes a two's compliment overflow (a<0 ==b<0) && (t<0 != a<0)

- Comparisons
	- ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/comparisons.png)
	-   cmp(b,w,l,q) S1, S2: Performs S2-S1 to set the above flags
	-  	test(b,w,l,q) S1 S2: Peforms S1&S2 to set condition flags

- **Set** sets the value of the specified register based on the associated conditions
	- ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/set_instructions.png)
- Program Flow Changers:
	- The jump commands go to a specific label
	-   ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/jump_instructions.png)
	- We can also move based on a specific conditions:
	- ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/move_instructions.png)
-   Jump Tables: Tables that have code location labels. The machine code can then use a register to find the corresponding location of code to jump to and execute.
-   The stack
	-   All information regarding the local data for a function gets stored on the sack
	- ![Credit:   
Computer Systems A Programmers Perspective 3rd Edition](https://raw.githubusercontent.com/Kdoje/CS4401-notes/master/assets/machine-code/stack_diagram.png)


