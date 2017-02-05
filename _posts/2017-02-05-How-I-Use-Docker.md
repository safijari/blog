---
layout: post
title: How I use Docker for development
---

Are you, dear reader, a little bit like me? 

Relatively comfortable with Linux but don't have them _mad l33t haxxor skillz_?

Do you ever find yourself getting frustrated when your OS yells 
*package _ihateyou_ is needed but is not going to be installed because F\*\*\* YOU THAT'S WHY....*

Have you ever run into a scenario where you are working on more than one project and each has 
a list of conflicting dependencies, and there may be ways to solve your problems by maybe 
compiling things from source or hacking your makefiles but you're just soooo very tired.

Finally, are you maybe a roboticist that either doesn't want to be tied down to Ubuntu 
for using ROS or is (rightfully) weary of the dependency hell it creates? 
(hint: try using OpenCV3 with ROS Indigo ... I dare ya!)

If some combination of the above is true, Docker may be for you! 
Not the _deploy your web app to the cloud and make it easy to scale_ variety, 
not even the _you want to use tensorflow? run *docker run tensorflow-container-
that-writes-your-thesis-for-you-and-makes-you-coffee_ variety, but the plain and simple 
_use containers like VMs but with barely any overhead and the ability to use your 
GPU to the fullest while not having to manage multiple environments much_ variety. 
Okay, that may actually not be that plain and simple but after reading this article 
you should be able to do just that!

## Before we begin, know that:

1. This article assumes that you have installed Docker, added your user to the docker group, 
and are familiar with some basic commands (e.g. run, ps, exec, start, stop, rm, rmi, attach). 
I will explain some of these in this article but past familiarity would be better.

2. The experience of using Docker is not going to perfectly like running VMs. 
You may run into some issues that you wouldn't in VirtualBox, but the net gain 
in my opinion is worth a few hiccups from time to time.

3. This may actually be a perfectly horrible way to use Docker. If you are a Docker 
veteran and see obvious flaws (especially security related) please let me know in the comments.

4. I use zsh with oh-my-zsh and a custom theme that, among other niceties, helps me avoid 
a confusion that can arise from using docker in this fashion namely _is a terminal I'm 
looking at in a container or not!_. I fix this issue by passing a special environment 
variable to the container when it is first run and then displaying it at my terminal 
prompt (effectively the Docker equivalent of Inception's _spinny top test_).

## Setting up the base Dockerfile

I recently started working with Nativescript and I needed to set up my development environment.

distro is Ubuntu and at Simbe Robotics we stick to the LTS versions. As such I will show 
use an Ubuntu 16.04 container as an example, though pretty much everything I show
here can be translated to other distributions. In case you missed orientation day:

`
``A Dockerfile is a simple configuration file that defines how your container is built. You can specify a base container, RUN a series of shell commands, set up environment variables, and specify what command runs when your container is started``
