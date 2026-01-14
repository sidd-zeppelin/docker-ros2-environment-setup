# ROS 2 Humble in Docker — A From-First-Principles Guide

This repository describes how to build a fully isolated, reproducible ROS 2 Humble
environment using Docker. The goal is not just to “install ROS”, but to understand
where every binary, key, repository, and dependency comes from.

What you end up with is a frozen robot operating system:
Linux + ROS + DDS + your code, portable to any machine.

────────────────────────────────────────────────────────────────────────────

1. WHY DO WE USE DOCKER FOR ROS?

ROS is not a normal application. It is a distributed middleware system that depends on:

- the Linux C library (glibc)
- the C++ ABI (libstdc++)
- Python ABI
- DDS networking
- the kernel clock and networking stack

If any of these change (Ubuntu upgrade, Python update, driver install),
ROS nodes start crashing in unpredictable ways.

Docker solves this by freezing the entire user-space OS.

Your computer runs:
  Linux kernel
    → Docker
       → Ubuntu 22.04
            → ROS 2 Humble
                 → Your robot software

Only the kernel is shared. Everything else is isolated and deterministic.

This is how professional robots are deployed.

────────────────────────────────────────────────────────────────────────────

2. WHAT DOCKER REALLY IS

Docker is not a virtual machine.

It does not emulate hardware.
It uses Linux kernel features to isolate:

- filesystem
- processes
- networking
- users
- environment variables

Each container thinks it is its own Linux computer, but it runs at native speed.

This makes Docker ideal for robotics, because ROS needs direct access to
networking, clocks, and devices.

────────────────────────────────────────────────────────────────────────────

3. WHY UBUNTU 22.04 IS REQUIRED

ROS 2 Humble is built against Ubuntu 22.04 (Jammy).

The ROS build farm compiled Humble using:

- glibc from Ubuntu 22.04
- GCC 11
- Python 3.10
- DDS libraries built for Jammy

If you try to run Humble on a different OS version, you get:
- ABI mismatch
- DDS crashes
- Python module failures

Therefore our Docker image must use Ubuntu 22.04.

────────────────────────────────────────────────────────────────────────────

4. WHERE ROS BINARIES COME FROM

ROS packages are not downloaded from GitHub.

They come from the ROS build farm:

  https://packages.ros.org

This is run by Open Robotics, the organization that owns ROS.

They:
- build ROS for each Ubuntu release
- test everything
- cryptographically sign the binaries
- publish them as .deb packages

This is exactly how Ubuntu packages are distributed.

────────────────────────────────────────────────────────────────────────────

5. HOW CRYPTOGRAPHIC TRUST WORKS

Ubuntu will only install packages that are cryptographically signed.

ROS provides its public signing key here:
  https://github.com/ros/rosdistro

This key is installed into:
  /usr/share/keyrings/ros-archive-keyring.gpg

When apt downloads ROS packages, it verifies that:
- the package was signed
- the signature matches Open Robotics

This prevents malicious robot software from being installed.

────────────────────────────────────────────────────────────────────────────

6. BUILDING THE DOCKER IMAGE

Create a file called Dockerfile:

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=en_US.UTF-8

RUN apt update && apt install -y \
    locales \
    curl \
    gnupg \
    lsb-release \
    software-properties-common \
    build-essential

RUN locale-gen en_US en_US.UTF-8
RUN update-locale LANG=en_US.UTF-8

RUN curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    | gpg --dearmor -o /usr/share/keyrings/ros-archive-keyring.gpg

RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
http://packages.ros.org/ros2/ubuntu jammy main" \
> /etc/apt/sources.list.d/ros2.list

RUN apt update && apt install -y \
    ros-humble-desktop \
    python3-colcon-common-extensions

RUN echo "source /opt/ros/humble/setup.bash" >> /root/.bashrc

WORKDIR /root

────────────────────────────────────────────────────────────────────────────

7. BUILD THE IMAGE

docker build -t ros2-humble .

This creates a reusable robot operating system image.

────────────────────────────────────────────────────────────────────────────

8. RUN THE CONTAINER

docker run -it --name ros2_container ros2-humble

Inside the container:

ros2

If you see command output, ROS is running.

────────────────────────────────────────────────────────────────────────────

9. CREATING A ROS WORKSPACE

mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
source install/setup.bash

The workspace is layered on top of ROS.
Your code never touches /opt/ros.

────────────────────────────────────────────────────────────────────────────

10. WHAT COLCON DOES

colcon is ROS’s build system.

It:
- reads package.xml
- resolves dependencies
- builds C++ and Python
- installs binaries
- generates ROS environment hooks

It is equivalent to cmake + make + pip, but ROS-aware.

────────────────────────────────────────────────────────────────────────────

11. SAVING THE ENVIRONMENT

After everything is installed:

docker commit ros2_container ros2-humble-ready

Now you can run the full robot OS anywhere:

docker run -it ros2-humble-ready

This gives you a complete ROS 2 Humble environment in seconds.

────────────────────────────────────────────────────────────────────────────

12. WHAT YOU HAVE BUILT

You now own a:

- cryptographically trusted
- ABI-stable
- DDS-enabled
- ROS 2 Humble
- Dockerized robot OS

This is the same architecture used by:
- autonomous vehicles
- drones
- industrial arms
- research robots
