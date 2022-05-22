# visualization_container

## Introduction

The goal of this repository is to be a reference to running a container built from a Dockerfile distributed via K3S to be able to control the display of the child node. The assumption is that the target node is Linux-based.

To test, we are going to play VLC in the container on the node, with the VLC output showing on the target node screen.

This is going to be building up to that step-by-step as I figure out how to do it.

## Why would we want to do this?

VLC actually has a telnet interface. We could have VLC playing on the host, and open the remote connection from within the container and control the player that way. So why not?

The problem is that VLC is on the target node itself then, and we lose most of our control. If VLC crashes we cannot do anything about it, and down the line if we want to replace VLC with a different program to drive the screen then we would need to go to every node running this, deactivate VLC, install the new program and figure out the interface to it.

With scale and the distribution Kubernetes provides we really want to keep as much containerized as possible.

## Content

1. [Setting Up xhost And $DISPLAY](#1-setting-up-xhost-and-display)
2. [Installing Docker on the target node](#2-installing-docker-on-the-target-node)
3. [Building the base Dockerfile](#3-building-the-base-dockerfile)
5. [Running the Docker image](#)
6. [Installing K3S](#)
7. [Building the deployment](#)
8. [Deploying the project](#)

## 1. Setting Up `xhost` And `$DISPLAY` 
> NOTE: This section was updated last at 2022/05/21

If the PC you want to display stuff on is at `192.168.1.39`, with user `user`, then run
```bash
# SSH to PC, enter password of PC you are accessing if prompted
$ ssh user@192.168.1.39
# Check if an display is available
$ echo $DISPLAY
# If you see something like `:0` then continue. If it is a blank link uncomment and run the line below:
# export DISPLAY=":0"
# Disable xhost access control. Ideally you would be more specific.
$ xhost +
# Test by running some application that has a visual interface
$ firefox
```
After following these steps you should have a window open on the PC you are connected to. If not, something went wrong and you will have to google.

## 2. Installing Docker on the target node
> NOTE: This section was updated last at 2022/05/21

Follow the official guide for your target node OS. For testing and development we are going to be doing that directly on the target node so that we are sure every step along the way works, and we can just build on that going forward.

You can find the Ubuntu guide [here](https://docs.docker.com/engine/install/ubuntu/)

## 3. Building the base Dockerfile
> NOTE: This section was updated last at 2022/05/22

For a start we are going to be making a dockerfile from the official Ubuntu 20.04 image from dockerhub, install VLC and then simply launch VLC. You can see this in [Dockerfile.base](/Dockerfile.base), but here is a breakdown:

```docker
# From the base ubuntu 20.04 image
FROM ubuntu:20.04
# Update the store and install VLC.
#   `-y` is important as VLC is large, so it will ask if you want to continue
#   `--no-install-recommends` blocks the installation of stuff we don't want to keep the image somewhat smaller.
RUN apt-get update && apt-get -y --no-install-recommends install vlc
# Create a new user, called `vlcuser`, to start VLC. VLC does not start as root so we need a proxy.
RUN useradd -ms /bin/bash vlcuser
# Switch from `root` to our new user
USER vlcuser
# Operate from this new user's directory
WORKDIR /home/vlcuser
# Run the VLC program. The options below are stuff VLC allows us to set when starting the program from the terminal
#   `--fullscreen`          -> Maximize the window
#   `--repeat`              -> Keep on playing the videos in the playlist. Otherwise it will stop playing.
#   `--video-on-top`        -> Display this window above anything else. This is important as we want our
#                              display to bewhat the end user sees.
#   `--no-video-title-show` -> Disable showing the title of the video when it starts playing.
#   `--no-audio`            -> Disable audio. We are not mounting any audio output in, so it will log issues without this.
#   `--no-qt-privacy-ask`   -> When launching VLC for the first time, it has a privacy protection pop-up. As "the first time"
#                              will be every time this container is created, we want to disable that.
#   /home/vlcuser/Videos    -> Add any videos inside this folder to the playlist. We are mounting this from the host so that
#                              some other container could change the videos and VLC will keep on doing its thing.
CMD [ "vlc", "--fullscreen", "--repeat", "--video-on-top", "--no-video-title-show", "--no-audio", "--no-qt-privacy-ask", "/home/vlcuser/Videos" ]
```

We can build this dockerfile on the target node with
```bash
$ scp Dockerfile.base user@internal_ip:~/Desktop
$ ssh user@internal_ip
$ cd ~/Desktop
$ docker build -f Dockerfile.base -t visualization_container:0.0.1 .
```

To run this image we need to do some mounting magic. First, get a test video and put in in the `~/Videos` folder on the target node. This is going to be our video we expect to see playing. We want to di it this way, as VLC will track this folder. Should we ever want to change the video, some other container can also have this folder mounted and simply swap out the video and VLC is none the wiser.

```bash
$ scp test_vid.mp4 user@internal_ip:~/Videos
```

Finally we can start the container. We have to set the $Display inside the container to match what is happening on the host, and we need to mount the X11 domain socket into the container as well so that we can access the host screen via the X11 protocol.

```bash
$ docker run \
             --env="DISPLAY=$DISPLAY" \
             --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
             -v ~/Videos:/home/vlcuser/Videos \
             visualization_container:0.0.1
```
and __voila!__, we see a video playing on the target PC. The dockerfile explains itself with some comments, but most of the magic is in the way we start VLC.

> NOTE: For a guide to adding sound, check out [this repository](https://github.com/TheBiggerGuy/docker-pulseaudio-example/blob/master/Dockerfile)

