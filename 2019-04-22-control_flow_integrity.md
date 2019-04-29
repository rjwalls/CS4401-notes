---
title:  "Lecture Notes: Basics of Control-flow Integrity"
date:   2019-04-22 09:00:00
categories: notes lecture 
layout: post
---

So far we have seen a number of software defenses (canaries to detect
overflows, non-executable memory to prevent code injection, ASLR to make
code-reuse difficult, etc.) and a number of ways to break those defenses (e.g.,
leveraging information leaks). Today, we are going to talk about a new class of
defenses based on the idea of **control-flow integrity (CFI)**.   
Control-flow integrity was first proposed in 2005 in the paper: "Control-Flow
Integrity: Principles, Implementations, and Applications." Abadi et al.,
CCS'05.

So how is CFI different from the other defenses we have been talking about?
The intuition behind CFI is relatively straight-forward: If we see deviations
between how the program is supposed to behave and how it actually behaves, then
we know something bad is happening and we can halt the program.  For this to
work we need three things:
 1. A definition of what the program should be doing.
 2. An efficient way of checking if behavior is expected.
 3. Mechanisms to keep the check in Step 2 secure. 

To make things efficient, CFI defenses limit the scope of "behavior that needs
to be checked" to only include control-flow transfers. To keep things simple,
we define a control-flow transfer as any point in the program that a value is
loaded from memory into the instruction pointer. For example, when the program
calls a function, that counts as a control-flow transfer. Similarly, when the
program returns from a function, that is a control-flow transfer. When an
attacker overwrites a return address, they are manipulating a control-flow
transfer. In a nutshell, CFI defenses check control-flow transfers at runtime
to make sure the targets (e.g., the location a function is returning to) are
legal. 


### Modeling Program Behavior

CFI-based defenses use a **control-flow graph (CFG)** to model the legal
execution of the program. The CFG models all legal control transfers between
functions.  Put simply, the CFG is a definition of what functions can call what
other functions.  Consider the following program for sorting two lists of
integers.


```c

sort2(int a[], int b[], int len)
{
  sort(a, len, lt);
  sort(b, len, gt);
}


bool lt(int x, int y){
  return x<y;
}

bool gt(int x, int y){
  return x>y;
}
```

Let's draw out what functions call which other functions. 

```

sort2 ->sort-->lt
            |->gt

```

This drawing is a decent start to our control-flow graph, but it is missing a
number of important details. For example, there is no notion of function
returns. In particular, we need some means to encode the behavior that the
first call to sort will return  to a different location than the second call to
sort.  To encode this behavior, we actaully need to break up our functions into
**basic blocks**. Think of a basic block as a sequence of machine instructions
that end with a jump or return or some other kind of control transfer. See the
figure below.

```

 sort2()
+--------------+                                            lt()
|              |                                    +---> +--------------+
|              |               sort()               |     |              |
+--------------+ +----------> +--------------+      |     |              |
                              |              |      | +-- +--------------+
+--------------+      ^-----> |              |      | |
|              | <------+     +--------------+ -----+ |
|              |      | |                           | |
+--------------+ +----+ |     +--------------+ <------+     gt()
                        |     |              |      |---> +--------------+
+--------------+        +-----+              | <--------- |              |
|              | <----------- +--------------+            |              |
|              |                                          +--------------+
+--------------+


```

Let's step through this graph.

```
sort2 -> sort -> lt -> sort -> sort2 -> sort -> gt -> sort -> sort2
```

So this figure represents our control-flow graph and acts as a model of process
behavior. Computing the CFG can either be done during compilation or directly
from the binary. The former is much more straight-forward than the latter, but
for these notes we won't go into any more details about how to compute the CFG.


### Checking Program Execution Against the Model

So how do CFI defences check compliance with the control-flow graph at
run-time?  The basic idea is to insert **instrumentation** into the binary.
This instrumentation is simply a few additional instructions inserted at
important locations in the code---for CFI these points are control-flow
transfers. Sometimes you'll hear this idea referred to as inline reference
monitors.

Actually, CFI defenses don't need to modify all control-flow transfers.
Considering our previous example, there is no need to monitor the direct calls
`sort2 -> sort (x2)` because the addresses are stored in the text segment  and
can't be changed (because the pages in the text segment are marked as
read-only). The defense would  have to monitor the call to the function pointer
and all function returns.

Specifically, CFI defenses will  add instrumentation  to check **indirect
branches**, i.e., any change in control flow that uses a dynamic value.
 - indirect jumps (jumps via register or memory operand)
 - indirect calls (calls via ...) 
 - return instructions



#### Forward-Edge Checks

To protect a forward-edge (i.e., indirect call), CFI  will:
 1. insert  a **label** just before the target address, and 
 2. insert instructions at the callsite to the label of the target, aborting if
    the label doesn't match.  The labels are determined by the CFG.

Simplest way to assign labels: make all labels the same. This approach is often
called **coarse-grained CFI**.

```
 sort2()
+--------------+                                            lt()
|              |                                    +---> +--------------+
|              |               sort()               |     |label l       |
+--------------+ +----------> +--------------+      |     |              |
                              |              |      | +-- +--------------+
+--------------+      ^-----> |              |      | |
|      label l | <------+     +--------------+ -----+ |
|              |      | |                           | |
+--------------+ +----+ |     +--------------+ <------+     gt()
                        |     |       label l|      |---> +--------------+
+--------------+        +-----+              | <--------- |label l       |
|      label l | <----------- +--------------+            |              |
|              |                                          +--------------+
+--------------+

```


Coarse-grained labeling tends to be vary efficient and easy to implement, but
it lacks precision. For instance, this labeling scheme would allow the `sort()`
function to return to the start of `lt()` or `gt()`. 

To address this issue, we can use a **fine-grained** labeling scheme, where we
use many different labels.  However, we want to avoid making the scheme too
complicated or the checks will take too long to run. Consider the example CFG
with fine-grained labeling.

```
 sort2()
+--------------+                                            lt()
|              |                                    +---> +--------------+
|              |               sort()               |     |label M       |
+--------------+ +----------> +--------------+      |     |              |
                              |              |      | +-- +--------------+
+--------------+      ^-----> |              |      | |
|      label l | <------+     +--------------+ -----+ |
|              |      | |                           | |
+--------------+ +----+ |     +--------------+ <------+     gt()
                        |     |       label N|      |---> +--------------+
+--------------+        +-----+              | <--------- |label M       |
|      label l | <----------- +--------------+            |              |
|              |                                          +--------------+
+--------------+


```

You'll notice that some of the targets still share the same label. We do this
to simplify the instrumentation. But how do we know what label to assign a
target? We will use the following constraints in this example:
 - Return sites from calls to sort must share label (L)
 - call targets gt and lt must share a label (M)
 - remaining labels are unconstrained (N)

Getting technical, CFI partitions the set of indirect branch targets into
**equivalence classes**. Two target addresses are equivalent if there is an
indirect branch that can jump to both targets according to the CFG. Each
equivalence class (and all of the target addresses in the class) is assigned an
**Equivalence-Class Number (ECN)**.  The **branch ECN** statically defines the
legal targets for a branch instruction. The **target ECN** is the ECN of the
actual destination at runtime. CFI says that the target ECN should be the same
as the branch ECN. 


#### The Instrumentation

At this point,  assume we have constructed the CFG, determined the equivalence
classes, and assigned each equivalence class a label. Now we have to add the
code (i.e., instrumentation) to enforce CFI.

Here's how the instrumentation for Classic CFI might look.
 1. First embed the ECN label at the target. The instruction you use doesn't
    matter as long as it is side-effect free and includes the ECN.
 2. Compare the target ECN to the branch ECN.  Steps: save the return address
    into a register. Compare the branch ECN to the target ECN (using the dummy
instruction.  Jump out if error.

```ASM

ret

####Rewritten to


#Code at the return address (i.e., target)
#Embedding the label

prefectchnta ECN


#Code at the branch

popq %rcx
compl $ECN, 4(%rcx)
jne error
jmpq *%rcx

```

#### Issue: Separate Compiliation

One of the primary drawbacks of Classic CFI is that it does not support
separate compilation. In other words, all application modules, including
libraries, have to be available at instrumentation time. Why? In part because
CFI (and its security properties) rely on an assumption of *global uniqueness*.

*global uniqueness assumption*: After CFI instrumentation, the bit patterns
chosen for the labels must not be present anywhere in the code memory except in
the labels and label checks. This includes opcode bytes. Because the
instrumentation is done at compile time. So you have to have all of the
libraries and the ECNs already instrumented into the library.

## Issue: Merging Equivalence Classes

While the use of equivalence classes makes each label check simple, the cost is
imprecision. In particular, this scheme will merge two sets of targets if they
aren't disjoint resulting in a loss of precision from the intended CFG.  For
example:

```
                  +-----+ <------+
                  |    1|        |
                  |     |        |
                  |     |        |
                  +-----+        |  +------+
                                 |  |      |
                  +-----+ <------+  |      |
                  |    2|        |  |      |
                  |     |        |  |      |
                  |     |        |  |      |
                  +-----+        |  |      |
                                 ^  |      |
         +------> +-----+ <------++ |      |
         |        |  3  |           +------+
         |        |     |
         |        |     |
+------+ |        +-----+
|      | |
|      | +------> +-----+
|      | |        |4    |
|      | |        |     |
|      | |        |     |
|      | |        +-----+
|      | |
|      +-v------> +-----+
+------+          |5    |
                  |     |
                  |     |
                  +-----+



```

In this scenario, targets 1,2,3,4, and 5 will all have the same equivalence class
even though 1,2 and 4,5 are reached from different indirect branches. 


#### Issue: Lack of Adoption


How common is control-flow integrity? On one hand, there are a ton of CFI
variations proposed by academia. On the other, classic CFI hasn't seen wide
adoption. Arguably this is for two primary reasons. First, because of the
performance overhead (16% on average, 45% worst case for classic CFI). Second,
because of the lack of support for  *separate compilation*. Classic CFI
requires all modules including library to be available at instrumentation time.
This implies that each binary must come with its own instrumented version of
libraries.

#### Issue: Soundness vs. Completeness for the CFG

It turns out that generating a control flow graph (CFG) that is both **sound**
and **complete** is an intractable problem. Instead, you must usually choose
between the two.  Intuitively, a complete analysis will not include any
branches that are not taken in the program. A sound analysis will always
accept a legal branch target.  It's the difference between missing edges and
having too many.

### Backward-Edge Checks

We could protect backward-edges (e.g., returns) using checks in a manner
similar to what we just described for forward-edges. However, relatively recent
work has found that forward-edge only protections are insufficient to protect
binaries given an called **control-flow juijutsu**.

#### Control-flow juijutsu 

It is important to keep in mind that CFI does not prevent an attacker from
overwriting the target of an indirect branch. Instead CFI makes it harder for
an attacker to exploit that bug by drastically limiting the number of legal
targets. As a consequence, as long as the attacker can craft an attack that
always jumps to legal labels, then CFI will allow the attack.

The researchers behind the control-flow juijutsu attack showed just that: in
practice, it is often possible to craft an attack that completely follows the
encoded CFG due to the presence of highly connected functions, called
**dispatcher functions**.  Dispatcher functions are  called by many other
functions in the code and thus have a lot of legal targets. It might help to
recall the imprecision issue we discussed earlier with respect to equivalence
class merging.  Printf is an example of such a dispatcher function.  Dispatcher
functions make it easy to get from any node in the CFG to any other node in the
CFG while only following legal branches---consequently making it easier to
expoit the program without running afoul of CFI.

#### The shadow stack

To address the challenge of dispatcher functions, it is necessary to add
additional precision to CFI in the form of a **shadow stack**.  The shadow
stack is a region in memory that is used to securely store return addresses.
The shadow stack allows a program, at runtime, to determine the correct return
address among all of the possible return targets. To implement the shadow
stack, we have to:
 - modify how the binary handles function prologues and epilogues to save the
   return address to the shadow stack.     
 - and protect the shadow stack from manipulation

Note, shadow stacks were proposed in the orginal CFI paper, but many later CFI
variations did not support them to improve performance.  



