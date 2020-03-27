---
title: "Lecture Notes: Getting Started with the Course Infrastructure"
date: 2020-03-26 01:01:00
categories: notes lecture 
layout: post
---

This video describes a simple local setup for solving challenge binaries. In
particular, it describes how to set up a local container for running binaries,
how to connect to the course shell server and access challenge binaries, and
how to submit flags.   


### Video!

<iframe width="560" height="315" src="https://www.youtube.com/embed/ncetH_pBTeg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


### Additional Tips

**Problem: Scrolling Repeated Text.** If you open epic treasure via docker and
have an issue (likely with tmux) where the text is printing incorrectly to the
terminal, try typing `exit` (and hitting enter, of course) to drop out of the
tmux session. 

**Problem: Windows and Docker.** Some students were able to get the docker
image working in Powershell with the following modified command:

```
docker run -v ${pwd}:/root/host-share --privileged -it --workdir=/root ctfhacker/epictreasure
```
