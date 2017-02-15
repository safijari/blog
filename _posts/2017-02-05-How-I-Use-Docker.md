---
layout: post
title: How I use Docker for development
excerpt_separator: <!--more-->
published: false
---

Are you, dear reader, a little bit like me? 

Relatively comfortable with Linux but don't have them _mad l33t haxxor skillz_?

Do you ever find yourself getting frustrated when your OS yells 
**package _ihateyou_ is needed but is not going to be installed because F\*\*\* YOU THAT'S WHY....**

Have you ever run into a scenario where you are working on more than one project and each has a list of conflicting dependencies, and there may be ways to solve your problems by maybe  compiling things from source or hacking your makefiles but you're just soooo very tired.

Finally, are you maybe a roboticist that either doesn't want to be tied down to Ubuntu  for using ROS or is (rightfully) weary of the dependency hell it creates?  (hint: try using OpenCV3 with ROS Indigo ... I dare ya!)

If some combination of the above is true, Docker may be for you! 

<!--more-->

Not the _deploy your web app to the cloud and make it easy to scale_ variety,  not even the _you want to use tensorflow? run **docker run tensorflow-container- that-writes-your-thesis-for-you-and-makes-you-coffee**_ variety, but the plain and simple _use containers like VMs but with barely any overhead and the ability to use your  GPU to the fullest while not having to manage multiple environments_ variety. Okay, that may actually not be that plain and simple but after reading this article you should be able to do just that!

#### Before we begin, know that:

1. This article assumes that you have installed Docker, added your user to the docker group, and are familiar with some basic commands (e.g. run, ps, exec, start, stop, rm, rmi, attach). I will explain some of these in this article but past familiarity would be better.

2. The experience of using Docker is not going to perfectly like running VMs. You may run into some issues that you wouldn't in VirtualBox, but the net gain in my opinion is worth a few hiccups from time to time.

3. This may actually be a perfectly horrible way to use Docker. If you are a Docker veteran and see obvious flaws (especially security related) please let me know in the comments.

4. I use zsh with oh-my-zsh and a custom theme that, among other niceties, helps me avoid a confusion that can arise from using docker in this fashion (namely _is a terminal I'm looking at running in a container or on the host!_). I fix this issue by passing a special environment variable to the container when it is first run and then displaying it at my terminal prompt (effectively the Docker equivalent of Inception's _spinny top test_).

#### Docker Essential Vocab

- **Image**: This is essentially the "installation" of something that you want to run using Docker. An image contains all the data necessary to run containers. Images are hierarchical and a new image that shares information with an older one will not reproduce this information and instead just re-use it (i.e. if you have two Ubuntu based images with different software installed, they will both refer to the same base Ubuntu image rather than copy its contents). This is what people mean when they say that Docker's filesystem is _layered_.

- **Container**: An instance of a particular image. This is equivalent to running, say Firefox, twice. In that scenario Firefox is installed once but can be launched multiple times and each instance refers back to the same installation. when a container is running, it can't make any changes to the underlying image (images are read only!) but gets assigned a new filesystem for storage of new information. Images may be **ephemeral** (that is they are removed once you stop them and any new data is destroyed), or **persistent** (containers are kept either in an on or off state until they are explicitly removed). Note that it is possible to mount external volumes onto the containers and all changes made to the host filesystem this way are persistent!

- **Dockerfile**: Simple configuration file that defines how your container is built. You can start FROM a base container, RUN a series of shell commands, set up ENVironment variables, ADD or COPY things from the host filesystem, specify what command runs when your container is started, and more! These files can start from scratch or from an existing image. Popular Linux distribution provide various images in an official capacity on ...

- **Docker Hub**: One of the online repository of Docker images that is set as the remote location as default. This is very similar to GitHub and can contain both public and private images. When you install Docker this is set up as the remote repository.

- **Naming/Tagging Images**: Typically when you run a docker container (e.g. `docker run ubuntu`), Docker will implicitly assume that you are looking for an image called ubuntu:latest. There are many other tags that Canonical has made for the Ubuntu Docker images: trusty, xenial, 14.04, 16.04, etc. Note that ubuntu:xenial and unbuntu:16.04 refer to the same actual image whereas ubuntu:trusty and ubuntu:xenial are completely different images underneath (with different Dockerfiles!). This allows for a way to pick the correct container while keeping things neatly organized under the same namespace.

#### Setting up the base Dockerfile

I recently started working with Nativescript and I needed to set up my development environment. Right off the bat I ran into a classic problem that Docker solves: I run Ubuntu 16.04 whereas the Nativescript installation guide recommends 14.04 (trusty). I built a container starting from  the trusty base image and adding all the software I needed on top, I will use the resulting Dockerfile as an example for this article. The full file can be found [here](https://github.com/safijari/docker-files/blablabla)(note that is must be named **Dockerfile** for the build process to work). I'll explain the constituent parts below:


```
FROM ubuntu:trusty
```

First we need to specify the base image. The FROM statement accomplishes this. When building this image Docker will first search for this image locally and if it fails it will then turn to a remote repository (e.g. Dockerhub). The tag `ubuntu:trusty` starts of with the base (and rather minimal) Trusty Tahr image.

```
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8
```

Similar to any other bare bones Linux setup, we need to run a command to choose a locale and set an environment variable to specify the language the system will use. The `RUN` command will, well, _run_ a command as root. Note that by default this command will run in `sh` rather than `bash` which, for the most part, doesn't make a difference. The `ENV` command sets a persistent environment variable where the syntax is `ENV <VAR> <VALUE>`.

```
RUN apt-get update && apt-get install -y \
    tmux \
    zsh \
    curl \
    wget \
    vim \
    sudo \
    libgl1-mesa-glx \
    libgl1-mesa-dri \
    mesa-utils \
    unzip \
    python-software-properties \
    software-properties-common \
    && rm -rf /var/likb/apt/lists/*
```

In this step I'm installing a bunch of _niceties_, things like `vim`, `zsh`, `tmux`, etc. `sudo` needs to be installed separately as well because by default Docker container images run their root user. There are ways to add a non-root user using `USERADD` but I will not be taking those steps and instead will mount the host user onto the container. And `software-properties-common` is needed to run the `apt-add-repository` command which makes adding a `ppa` easier.

Note the first and last lines. It is common practice to clean the sources list (last line) in a Docker container because in most situations it is not going to serve a purpose (once your container is ready you usually will not need to install new software in it). Because of this I had to fist run an `apt-get update` (first line) which is a good practice anyway.

If your computer is running an Intel GPU and you would like your container to be able to access it, you should also install `libgl1-mesa-glx`, `libgl1-mesa-dri`, and `mesa-utils`. You can then decide to use the GPU at runtime. Note that this is also possible to do with NVidia GPUs (I have never tried with AMD) and NVidia even puts out a utility called `nvidia-docker` which makes the process simpler. I use this with my desktop to run GPU powered Tensorflow!

You can have multiple commands in a single `RUN` statement but they need to be separated by `&&`. a `\` signals a line break and is purely cosmetic but is very useful in making Dockerfiles more readable (and easily editable).

Finally, notice the `-y` in the `apt-get install` statement. This is necessary and without the image cannot be built. This is because the Docker build process cannot stop for user input, instead the user input needs to be signaled in advance. The `-y` command tells `apt` to install all the packages without further user input.

```dockerfile
# Nativescript stuff from here https://docs.nativescript.org/start/ns-setup-linux
RUN apt-get update && apt-get install -y \
    lib32z1 \
    lib32ncurses5 \
    lib32bz2-1.0 \
    libstdc++6:i386 \
    g++ \
    && rm -rf /var/likb/apt/lists/* # to clean the sources list

# Need to break this up because of the add-apt stuff
RUN add-apt-repository ppa:webupd8team/java -y \
    && apt-get update \
    && apt-get install oracle-java8-installer

ENV JAVA_HOME $(update-alternatives --query javac | sed -n -e 's/Best: *\(.*\)\/bin\/javac/\1/p')
```

The next few commands in the Dockerfile are just installation of Nativescript dependencies and setting a required environment variable.

```dockerfile
COPY android-studio.zip /tmp/android-studio.zip
RUN unzip /tmp/android-studio.zip /opt/ && rm /tmp/android-studio.zip
COPY vscode.deb /tmp/vscode.deb
RUN dpkg -i /tmp/vscode.deb && rm /tmp/vscode.deb
```

And now for something new. The COPY command takes a file from the host filesystem and copies it into the Docker image's filesystem. Note that the docker build process only has access to the files in the directory that the Dockerfile resides in. I tend to organize my Dockefiles under `~/Docker/<image-name>/` and this way I can keep such dependencies that need to be copied organized as well. In this case the dependencies are the android studio and visual studio code. The former is required for Nativescript development and the latter is highly recommended. Neither are available in the repos. Note that I delete the `zip` and `deb` files from the image after they have been used since they are not needed at that point and just clutter up the image filesystem.

At this point the Dockerfile is complete and can be built. If you want to try this on your own, download the full file from **here**, download android studio from **here** and rename the zip file `android-studio.zip`, and download Visual Studio Code **here** and rename it `vscode.deb`. Put all of these files in the same folder, go to that folder in a terminal and type the following:

```
docker build -t nativescript-dev .
```

It is very easy to forget the dot (.) at the end of that command, but maybe looking at the command structure will help: the command essentially boils down to `docker build -t <image-name> <image-folder>`. Actually you don't even need the `-t` option that lets you give the image a name. Docker builds a new image for every `RUN`, `ENV`, `COPY` etc. command in the Dockerfile and assigns a hash to it. Each of these intermediate images depends on the image that was created as the result of the previous command. Any of these images can be run (or used in a `FROM` command) using their hash. They can even be tagged after the fact. The `-t` option applies a tag to the final such image created when the Dockerfile finishes building.

If this build command finishes successfully (and by all rights it should), then a new image called nativescript-dev will now exist in your local Docker repository. You can confirm this by running `docker images`.
