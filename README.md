# visualization_container

> NOTE: This README was updated last at 2022/05/21

The goal of this repository is to be a guide to running a container built from a Dockerfile distributed via K3S to be able to control the display of the child node. The assumption is that the target node is Linux-based.

As a stretch goal we are also going to attempt to run a Python OpenCV image and have the video feed be the output on the remote screen, __even when a USB camera is connected or removed during the container lifecycle.__

This is going to be building up to that step-by-step as I figure out how to do it.

**Steps:**
1. [Setting Up xhost And $DISPLAY](#)
2. [Installing Docker on the target node](#)
3. [Building the Docker Image content](#)
4. [Building the Dockerfile](#)
5. [Running the Docker image](#)
6. [Installing K3S](#)
7. [Building the deployment](#)
8. [Deploying the project](#)

## 1. Setting Up `xhost` And `$DISPLAY` 

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