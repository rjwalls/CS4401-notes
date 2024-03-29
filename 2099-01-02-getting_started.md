---
title: "Lecture Notes: Getting Started"
date: 2020-01-02 01:01:00
categories: notes lecture 
layout: post
---

This set of notes lists what you should do to prepare for the first lecture. It
also introduces the course infrastructure and describes the recommended setup
for solving challenge binaries. In particular, it describes how to set up a
local container for running binaries, how to connect to the course shell server
and access challenge binaries, and how to submit flags.


### Preparing for the Course

To prepare for the course, you should do the following before the first class (time permitting):

 1. Read the welcome email. The instructor should send this email a day or two before the first class. 
 2. Watch the [Tips and Tricks videos](https://youtube.com/playlist?list=PLeKxIn6N-kCi38WxOqNBXhxrZnOE9SVET). This playlist contains videos of previous students and covers what they think you should know before taking this course.
 3. Read the [course syllabus](https://cs4401.walls.ninja/syllabus). 
 4. Create an account on the [course infrastructure](https://cs4401.walls.ninja/). If you are reading this, then you've probably already completed this step.
 5. Join the course Discord Server using the link in the welcome email. In the future, I will use Discord rather than email for course communications. You should set your Discord nickname for the server as your real name.
 6. Look at the [schedule](https://cs4401.walls.ninja/schedule) for the first couple of lectures. 
 7. If time allows, read the notes below to set up your local environment for challenge solving.

Optional Preparaton:

 1. If you'd like an introduction (or refresher) to using command-line
    interfaces, I highly recommend completing the [Bandit
challenges](https://overthewire.org/wargames/bandit/). 

### Course Docker Container

I recommend solving most course challenges inside of an [Epic
Treasure](https://github.com/rjwalls/EpicTreasure) docker container. This
container comes with a variety of valuable tools preinstalled.  To use this
container, you'll first need to install docker. Afterward, you can run the
following command (or a variation) in your shell to launch the container. 

```
docker run --privileged -v ${PWD}:/root/host-share --rm -it --workdir=/root rjwalls/epictreasure tmux
```

This command will launch a [tmux](https://github.com/tmux/tmux/wiki) session.
For some Windows users, tmux has an issue with text printing incorrectly, e.g.,
scrolling repeated text. If you have such problems or prefer not to use tmux,
you can omit `tmux` from the previous command. 

**To disable ASLR in the docker container run the following after launching the
container.** Note: you'll need to run this command everytime you launch a new
Epic treasure container.

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

### Video: Setting up your Local Environment for Challenge Solving

The following video describes my recommended local setup for solving challenge
binaries. In particular, it describes how to set up a local container for
running binaries, how to connect to the course shell server and access
challenge binaries, and how to submit flags.

<iframe width="560" height="315" src="https://www.youtube.com/embed/hDTuJGkkG2c" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Setting Up pwntools and gef on the Course Server

The bulk of your challenge solving should be done in your local container.
However, it is often useful to be able to run pwntools and gef on the course
server as well. I've already setup pwntools for you. However, to set up GEF run the following on the course
server(s):

```bash
# Install gef
wget -O ~/.gdbinit-gef.py -q https://gef.blah.cat/py && \
    echo source ~/.gdbinit-gef.py > ~/.gdbinit && \
    python3 -m pip install ropper
```
