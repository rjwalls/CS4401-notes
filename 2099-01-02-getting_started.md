---
title: "Lecture Notes: Setting up your Local Environment for Challenge Solving"
date: 2020-01-02 01:01:00
categories: notes lecture 
layout: post
---

This video describes a simple local setup for solving challenge binaries. In
particular, it describes how to set up a local container for running binaries,
how to connect to the course shell server and access challenge binaries, and
how to submit flags.   


### Video!

<iframe width="560" height="315" src="https://www.youtube.com/embed/hDTuJGkkG2c" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Additional Tips

**Problem: Scrolling Repeated Text.** The epic treasure docker image starts you
in a tmux session. If you have an issue where the text is printing incorrectly
to the terminal, try typing `exit` (and hitting enter, of course) to drop out
of the tmux session. 

**Problem: Windows and Docker.** Some students were able to get the docker
image working in Powershell with the following modified command:

```
docker run -v ${pwd}:/root/host-share --privileged -it --workdir=/root ctfhacker/epictreasure
```
