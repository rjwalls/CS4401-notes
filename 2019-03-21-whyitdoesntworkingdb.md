---
title:  "Why Doesn't it work in GDB?"
date:   2019-03-21 09:00:00
categories: information 
layout: post
---

When executing code for smashing the stack GDB is incredibly helpful, but what about when your exploit works in GDB but doesn't work outside it.  One potential issue is that GDB sets two environment variables that don't exist outside of GDB, LINES and COLUMNS.

One solution to finding out what memory addresses will be outside GDB is to use GDB but just unset those two environment variables.

```
	gdb stack4

	unset env LINES
	unset env COLUMNS
```
Then you will be able to set breakpoints and examine the environment variables on the stack and find the "true" memory addresses.


