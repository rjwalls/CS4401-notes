---
title:  "Why Doesn't it work in GDB?"
date:   2019-03-21 09:00:00
categories: information 
layout: post
---


Stack addresses are going to be different in different execution environments. You might find that your exploit string works in GDB, but segfaults outside of GDB. Or maybe you got it working on your local machine but not on the server? This is because the stack addresses are different (sometimes just a little bit different) when you run in different environments. To understand why, remember that I said that environment variables are loaded onto the stack when you launch a process. If different environments have different values for those variables then the amount of stack space used to store the environment variables will be different. Consequently, the stack addresses further down in the stack will also differ. 

GDB sets two environment variables that don't exist outside of GDB, `LINES` and `COLUMNS`. We can just unset those two environment variables.

```
unset env LINES
unset env COLUMNS
```

Then you will be able to set breakpoints and examine the environment variables on the stack and find the "true" memory addresses.


