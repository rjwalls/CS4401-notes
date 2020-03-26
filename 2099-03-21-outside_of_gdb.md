---
title:  "Quick Notes: Help! My Exploit Works in GDB, but not on the Server?
date:   2099-03-30 01:01:00
categories: notes quick
layout: post
---

Stack addresses are going to be different in different execution environments.
You might find that your exploit string works in GDB, but segfaults outside of
GDB. Or maybe you got it working on your local machine but not on the server?
This is because the stack addresses are different (sometimes just a little bit
different) when you run in different environments. To understand why, remember
that I said that environment variables are loaded onto the stack when you
launch a process. If different environments have different values for those
variables then the amount of stack space used to store the environment
variables will be different. Consequently, the stack addresses further down in
the stack will also differ. 

In particular, GDB (at least some versions) sets two environment variables that
don't exist outside of GDB, `LINES` and `COLUMNS`. Sometimes, we can just unset
those two environment variables and the memory addresses will be the same
when running the process inside of and outside of GDB. 

```
unset env LINES
unset env COLUMNS
```



