---
title:  "Core Dumps"
date:   2019-03-13 08:00:00
categories: notes
layout: post
---



When working on CTF binaries you will often come across an error
message like the following: `Segmentation fault (core dumped)`. Turns out you
can use these core dumps with GDB to look at the state of memory when the
program crashed: `gdb ./ctfbinary coredump`. 

There is a big caveat here: even though that message is printed the actual
`core` file may not be saved (at least not anywhere that is accessible to you).  
If you are lucky, the core file will be saved to the current directory (so make
sure you have write permissions). If it is not there, check if `ulimit -c`
returns `0`. If so, change that value to something reasonable...like `ulimit -c
unlimited`. Be careful, core files can get pretty big so don't fill up the
competition server.     

If changing the limits doesn't work, it might be because some machines won't
dump the core for a `setuid` binary (e.g., Ubuntu 16.04). If that's the case,
you'll need to create a local copy of the binary and work on that. 

You can view how the kernel handles core dumps using:

```bash

$ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport %p %s %c %d %P
```  

Here `apport` is utility that handles the core dump and decides what to do with
it. If `setuid` is your problem than you will see the following messages the
`apport` logs (located at `/var/log/apport.log`)

```
ERROR: apport (pid 237154) Tue Mar 12 21:06:04 2019: called for pid 237153, signal 11, core limit 0, dump mode 2
ERROR: apport (pid 237154) Tue Mar 12 21:06:04 2019: not creating core for pid with dump mode of 2
ERROR: apport (pid 237154) Tue Mar 12 21:06:04 2019: executable: /problems/stack3r_50_98a7a69df0b744c649d095c0f6a39e64/stack3 (command line "./stack3")
ERROR: apport (pid 237154) Tue Mar 12 21:06:04 2019: executable does not belong to a package, ignoring

```

You can find more information using `man core` or [at this post about finding
core
dumps](https://askubuntu.com/questions/966407/where-do-i-find-the-core-dump-in-ubuntu-16-04lts),
[or at this
post](https://stackoverflow.com/questions/2065912/core-dumped-but-core-file-is-not-in-the-current-directory/),
[or
here](https://stackoverflow.com/questions/16048101/changing-location-of-core-dump),
and finally
[here](https://unix.stackexchange.com/questions/277331/segmentation-fault-core-dumped-to-where-what-is-it-and-why).
