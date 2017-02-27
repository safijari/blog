---
layout: post
title: How I use Docker for Robotics Development
excerpt_separator: <!--more-->
published: true
---

Are you, dear reader, a little bit like me? 

Relatively comfortable with Linux but don't have them _mad l33t haxxor skillz_?

Do you ever find yourself getting frustrated when your OS yells 
**package _ihateyou_ is needed but is not going to be installed because F\*\*\* YOU THAT'S WHY....**

Have you ever run into a scenario where you are working on more than one project and each has a list of conflicting dependencies, and there may be ways to solve your problems by maybe  compiling things from source or hacking your makefiles but you're just soooo very tired.

Finally, are you maybe a roboticist that either doesn't want to be tied down to Ubuntu  for using ROS or is (rightfully) weary of the dependency hell it creates?  (hint: try using OpenCV3 with ROS Indigo ... I dare ya!)

If some combination of the above is true, Docker may be for you! 

<!--more-->

I refer to not the _deploy your web app to the cloud and make it easy to scale_ variety,  not even the _you want to use tensorflow? just use **docker run tensorflow-container-that-writes-your-thesis-for-you-and-makes-you-coffee**_ variety, but the plain and simple _use containers like VMs but with barely any overhead and the ability to use your  GPU to the fullest while not having to manage multiple environments_ variety. Okay, this may actually not be _that_ plain and simple but after reading this article you should be able to do just that!

#### Before we begin, know that:

1. This article assumes that you have installed Docker, added your user to the docker group, and are familiar with some basic commands (e.g. run, ps, exec, start, stop, rm, rmi, attach). I will explain some of these in this article but past familiarity would be better.

2. The experience of using Docker is not going to perfectly like running VMs. You may run into some issues that you wouldn't in VirtualBox, but the net gain in my opinion is worth a few hiccups from time to time.

3. This may actually be a perfectly horrible way to use Docker. This setup has been designed for maximum convenience and security was the furthest thing from my min. If you are a Docker veteran and see obvious flaws (especially security related) please let me know in the comments.

4. I use zsh with oh-my-zsh and a custom theme that, among other niceties, helps me avoid a confusion that can arise from using docker in this fashion (namely _is a terminal I'm looking at running in a container or on the host!_). I fix this issue by passing a special environment variable to the container when it is first run and then displaying it at my terminal prompt (effectively the Docker equivalent of Inception's _spinny top test_).

#### Docker Essential Vocab

- **Image**: This is essentially the "installation" of something that you want to run using Docker. An image contains all the data necessary to run containers. Images are hierarchical and a new image that shares information with an older one will not reproduce this information and instead just re-use it (i.e. if you have two Ubuntu based images with different software installed, they will both refer to the same base Ubuntu image rather than copy its contents). This is what people mean when they say that Docker's filesystem is _layered_.

- **Container**: An instance of a particular image. This is equivalent to running, say Firefox, twice. In that scenario Firefox is installed once but can be launched multiple times and each instance refers back to the same installation. when a container is running, it can't make any changes to the underlying image (images are read only!) but gets assigned a new filesystem for storage of new information. Images may be **ephemeral** (that is they are removed once you stop them and any new data is destroyed), or **persistent** (containers are kept either in an on or off state until they are explicitly removed). Note that it is possible to mount external volumes onto the containers and all changes made to the host filesystem this way are persistent!

- **Dockerfile**: Simple configuration file that defines how your container is built. You can start FROM a base container, RUN a series of shell commands, set up ENVironment variables, ADD or COPY things from the host filesystem, specify what command runs when your container is started, and more! These files can start from scratch or from an existing image. Popular Linux distribution provide various images in an official capacity on ...

- **Docker Hub**: One of the online repository of Docker images that is set as the remote location as default. This is very similar to GitHub and can contain both public and private images. When you install Docker this is set up as the remote repository.

- **Naming/Tagging Images**: Typically when you run a docker container (e.g. `docker run ubuntu`), Docker will implicitly assume that you are looking for an image called ubuntu:latest. There are many other tags that Canonical has made for the Ubuntu Docker images: trusty, xenial, 14.04, 16.04, etc. You can be more explicit in which version you want by using something like `ubuntu:xenial` Note that `ubuntu:xenial` and `unbuntu:16.04` refer to the same actual image whereas `ubuntu:trusty` and `ubuntu:xenial` are completely different images underneath (with different Dockerfiles!). This allows for a way to pick the correct container while keeping things neatly organized under the same namespace.

#### Setting up the Dockerfile

Since [ros skillz pay my billz](twitter link) I, will focus on ROS development in this write up. The full file can be found [here](https://github.com/safijari/docker-files/blablabla) (note that is must be named **Dockerfile** for the build process to work). I'll explain the constituent parts below:

First we need to specify the base image. The `FROM` statement accomplishes this:

```
FROM ros:kinetic-desktop-full
```

 When building this image Docker will first search for this image locally and if it fails it will then turn to a remote repository (e.g. Dockerhub). The tag `ros:kinetic-desktop-full` starts this container off with a full desktop install of ROS Kinetic on the top of xenial. Note that if you were to look at the Dockerfile for our base image, you'll find that in includes a `FROM ros:kinetic-desktop`. In fact, you can go through the Dockerfiles for all such images you'll find the following hierarchy:

```
ros:kinetic-desktop-full 
   |
   --> ros:kinetic-desktop 
          |  
          --> ros:kinetic-base
                 |
                 --> ros:kinetic-core
                        |
                        --> ubuntu:xenial
```

Where the buck stops at `ubuntu:xenial`. If you look at the ubuntu xenial Dockerfile you'll find `FROM scratch`, which effectively specifies an empty image. People from Canonical then add to it an image of a very minimalistic ubuntu xenial. If you inspect the ROS Kinetic Dockerfiles, you will also finds lines like the ones below:

```
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
```

Similar to any other bare bones Linux setup, we need to run a command to choose a locale and set an environment variable to specify the language the system will use. The `RUN` command will, well, _run_ a command as root. Note that by default this command will run in `sh` rather than `bash` which, for the most part, doesn't make a difference. The `ENV` command sets a persistent environment variable where the syntax is `ENV <VAR> <VALUE>`.

During development I rely on a number of tools that are not going to be included in these images. So we need to install them separately:

```bash
RUN apt-get update && apt-get install -y \
    tmux \
    zsh \
    curl \
    wget \
    vim \
    emacs24 \
    sudo \
    libgl1-mesa-glx \
    libgl1-mesa-dri \
    mesa-utils \
    unzip \
    && rm -rf /var/likb/apt/lists/*
```

Most of these are just _niceties_, things like `vim`, `zsh`, `tmux`, etc. Note that unlike usual Docker setups, I do not like to use the root user in a container (I'll elaborate more on this later). As such, I like to install `sudo` which is normally not part of base Docker images for any distro. There are ways to add a non-root user using `USERADD` but I will not be taking those steps and instead will mount the host user onto the container. 

Now let's focus on the first and last lines. It is common practice to clean the sources list (last line) in a Docker container because in most situations it is not going to serve a purpose (once your container is ready you usually will not need to install new software in it). Because of this I had to fist run an `apt-get update` (first line). This is actually kind of the point. since a container may have been built in the past and any stored information about the repositories would likely be out of date. As such removing this information forces users to fetch the data first which is a good practice anyway.

If your computer is running an Intel GPU and you would like your container to be able to access it, you should also install `libgl1-mesa-glx`, `libgl1-mesa-dri`, and `mesa-utils`. You can then decide to use the GPU at runtime. Note that this is also possible to do with NVidia GPUs (I have never tried with AMD) and NVidia even puts out a utility called `nvidia-docker` which makes the process simpler. I use this with my desktop to run GPU powered Tensorflow and have even run containerized video games through steam in the past.

As should be evident from the above command, it is possible to have multiple commands in a single `RUN` statement but they need to be separated by `&&`. A `\` signals a line break and is purely cosmetic but is very useful in making Dockerfiles more readable (and easily editable).

Finally, notice the `-y` in the `apt-get install` statement. This is necessary and without the image cannot be built. This is because the Docker build process does not stop for user input, instead the user input needs to be signaled in advance. The `-y` command tells `apt` to install all the packages without further user input.

At the end of a Dockerfile, we can specify a command that should run when the container is started. For this container I have specified `zsh` as that command, as I would like the container to drop me into a shell at run time.

```
CMD ["zsh"]
```

### Building the Image

Just run 

```bash
docker build -t kinetic:dev .
```

It is very easy to forget the dot (.) at the end of that command, but maybe considering the command structure will help: the command essentially boils down to `docker build -t <image-name><:optional-version-tag> <image-folder>`. Actually you don't even need the `-t` option that lets you give the image a name. Docker builds a new image for every `RUN`, `ENV`, `COPY` etc. command in the Dockerfile and assigns a hash to it. Each of these intermediate images depends on the image that was created as the result of the previous command. Any of these images can be run (or used in a `FROM` command) using their hash. They can even be tagged after the fact and a single image can have multiple tags. The `-t` option applies a tag to the final such image created when the Dockerfile finishes building.

Note that if you created another Dockerfile that depended upon `ros:kinetic-desktop-full`, it would not download another copy of the base image. It wouldn't even make a new copy of that image on disk. This is the power of Docker's `layered` file system. You can even run multiple instances of containers that all refer back to these images

If this build command finishes successfully (and by all rights it should!), then a new image called  will now exist in your local Docker repository. You can confirm this by running `docker images`.

[SHOW OUTUPUT]


### The Run Script

The docker CLI program exposes a `run` command to the user. At its simplest, you can use it as `docker run <container-name>`. This by itself will only work for applications that are not interactive and do not need to have a `tty` allocated. Neither of these are true for the command we set earlier. Furthermore, this command runs Docker under a root user, does not do any X-forwarding (so no GUI apps), does not give it access to the GPU, and so on and so forth. I tend to use the following bash script that runs the container with all options that I normally need:

```bash
#!/bin/bash
xhost +local:
docker run -it --net=host 
  --user=$(id -u) \
  -e DISPLAY=$DISPLAY \
  -e QT_GRAPHICSSYSTEM=native \
  -e CONTAINER_NAME=cv-opensfm-dev \
  -e USER=jari \
  --workdir=/home/$USER \
  -v "/tmp/.X11-unix:/tmp/.X11-unix" \
  -v "/etc/group:/etc/group:ro" \
  -v "/etc/passwd:/etc/passwd:ro" \
  -v "/etc/shadow:/etc/shadow:ro" \
  -v "/etc/sudoers.d:/etc/sudoers.d:ro" \
  -v "/home/$USER/:/home/$USER/" \
  --device=/dev/dri:/dev/dri \
  --name=cv-opensfm-dev \
  cv-opensfm:imported \
```

There is a lot going on in that script, so let's break it down. The first line 

```bash
xhost +local:
```

Is required for the container to be able to create graphical windows on the host, so if you want to use any GUI applications (e.g. `rviz`). Note that this may NOT be the best or most secure way! As I said before, if you know a better way kindly chime in below :relaxed:. The next statement is the full `docker run` command. We'll take in chunks:

```bash
docker run -it --net=host 
```

This section ensures that the container runs in interactive mode with a tty allocated (`-it`) which is needed if you want to be able to use a terminal properly. `--net=host` makes it so that programs running in the container can access the host's network directly. I like to run ROS containers in this mode since it allows me to keep the same workflow I would have if I had ROS installed natively (i.e. a local ROS master would be accessible at localhost:11311). Pointing the container at a master running elsewhere on your network (e.g. on your robot) also becomes trivial. 

```bash
  --user=$(id -u) \
```

This line tells Docker to use a user with the same ID as your current user on the host. Typically this will just be 1000 if you have a single user machine. After this point, we pass a number of runtime environment variables to the container. Note that these will be persistent for the lifetime of the container. For this I use the `-e <container_var>:<host_var>` flag.

```bash
  -e DISPLAY=$DISPLAY \
```

The first environment variable is useful for telling Docker which display to use. 

```bash
  -e QT_GRAPHICSSYSTEM=native \
```

The second variable `QT_GRAPHICSSYSTEM` is necessary to set because of a weird bug I came across with QT windows and Docker. Since ROS GUIs are all QT based it's a good idea to set this! The `CONTAINER_NAME` variable

```bash
  -e CONTAINER_NAME=cv-opensfm-dev \
```

This is a special environment variable my zsh profile uses in order to show if the terminal window I'm looking at is in a container and what its name is.

```bash
  -e USER=$USER \
```

Many Linux scripts will look for the `$USER` variable to find the user directory so it's useful to set this as well.

```bash
  --workdir=/home/$USER \
```

The `--workdir` flag tells Docker which folder to always start a command in. In this case the _command_ will be `bash` or `zsh` so it makes sense for them to open in the user's home directory. Next I mount a bunch of directories using the flag `-v <contianer_folder>:<host_folder>:<optional_permissions>`.

```bash
  -v "/tmp/.X11-unix:/tmp/.X11-unix" \
  -v "/etc/group:/etc/group:ro" \
  -v "/etc/passwd:/etc/passwd:ro" \
  -v "/etc/shadow:/etc/shadow:ro" \
  -v "/etc/sudoers.d:/etc/sudoers.d:ro" \
  -v "/home/$USER/:/home/$USER/" \
```

The first one basically gives Docker access to the Unix domain socket for X11 so that it can open GUIs on the host. The other lines mount groups, sudoers, as well as the actual home directory. Note that if you're not comfortable with mounting your entire home folder for whatever reason, you can cherry pick but many benefits of doing this (e.g. consistent dotfiles/shell history across host and continer) will disappear.


