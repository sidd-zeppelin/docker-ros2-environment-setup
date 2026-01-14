### ROS 2 Humble in Docker — A From-First-Principles Guide
- A guide by https://github.com/sidd-zeppelin

This repository helps you create ROS2 Humble environment in any system and how to save it to use it in other different systems.
Text editor installed in the environment: neovim

---
## If you wanna skip this tutorial and just setup the environment, pull my ros2 environment docker file:

```
docker pull siddzeppelin/ros2-humble-sidd-zeppelin
```
and run to get the ros2 humble environment:
```
docker run -it siddzeppelin/ros2-humble-sidd-zeppelin
```

You have a running docker environment.
You now have a ROS2 Humble system on the new machine.


---

This repository explains how to build a clean, isolated, and reusable ROS 2 Humble
environment using Docker.

The goal is not just to “install ROS”, but to understand:
- where ROS comes from
- how Ubuntu gets ROS packages
- how cryptographic trust works
- and how everything is frozen into a portable robot OS

In the end, you will have:

Linux + ROS 2 + DDS + your code  
all inside a Docker image that can run anywhere.

---

## 1. Why we use Docker for ROS

ROS is not just a program. It depends on many low-level things:

- Linux system libraries (glibc)
- C++ runtime
- Python version
- DDS networking
- system clocks and networking

If any of these change (Ubuntu upgrade, Python update, driver install), ROS can
break in strange ways.

Docker solves this by freezing the whole Linux environment.

Your system becomes:
```
Linux kernel
    → Docker
    → Ubuntu 22.04
    → ROS 2 Humble
    → Your robot code
```

Only the kernel is shared. Everything else is locked in place.

This is how professional robots are deployed.

---

## 2. What Docker really is

Docker is not a virtual machine.

It does not pretend to be new hardware.
Instead, it uses Linux to isolate:

- files
- running programs
- networks
- users
- environment variables

Each container feels like its own Linux computer, but it runs at full speed.

This is perfect for ROS, which needs real networking and real timing.

---

## 3. Why Ubuntu 22.04 is required

ROS 2 Humble was built for Ubuntu 22.04 (Jammy).

The ROS build servers used:
- glibc from 22.04
- GCC 11
- Python 3.10
- DDS libraries for Jammy

If you use another OS version, things will break:
- DDS may crash
- libraries may not match
- Python modules may fail

So the Docker container must use Ubuntu 22.04.

---

## 4. Where ROS packages come from

ROS packages do not come from GitHub.

They come from the ROS build farm:
```
https://packages.ros.org/
```

This is run by Open Robotics, the group that maintains ROS.

They:
- build ROS for each Ubuntu version
- test everything
- sign the packages
- publish them as `.deb` files

This works the same way Ubuntu publishes its own packages.

---

## 5. How trust works

Ubuntu only installs packages that are signed.

ROS provides its public signing key here:
```
https://github.com/ros/rosdistro
```

This key is saved on your system at:
```
/usr/share/keyrings/ros-archive-keyring.gpg
```


When Ubuntu downloads a ROS package, it checks:
- was it signed?
- does the signature match Open Robotics?

If not, the package is rejected.

This protects your robot from bad or fake software.

---

## 6. Building the ROS Docker environment

First, download Ubuntu 22.04:

```
bash
docker pull ubuntu:22.04
```
Now start a fresh container:
```
docker run -it --name {container_name} ubuntu:22.04
```
You are now inside a new Linux system.

Update package lists:
```
apt update
```

Install basic system tools:
```
apt install -y locales curl gnupg lsb-release software-properties-common build-essential neovim
```
These packages do:

- `locales` – enables UTF-8 (ROS needs this)
- `curl` – downloads files
- `gnupg` – checks signatures
- `lsb-release` – tells Ubuntu its version name
- `software-properties-common` – lets us add new package sources
- `build-essential` – compiler and build tools
- `neovim` - text editor to run inside the container

Enable UTF-8:
```
locale-gen en_US en_US.UTF-8
update-locale LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```
Add ROS signing key:
```
curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
 | gpg --dearmor -o /usr/share/keyrings/ros-archive-keyring.gpg
```
Tell Ubuntu where ROS packages live:
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" \
> /etc/apt/sources.list.d/ros2.list
```
Update package lists again:
```
apt update
```
Install ROS 2 Humble:
```
apt install -y ros-humble-desktop
```
Install the ROS build system:
```
apt install -y python3-colcon-common-extensions
```
Enable ROS in every shell:
```
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source /opt/ros/humble/setup.bash
```
Test:
```
ros2
```

---

## 7. Create a ROS workspace
```
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build
source install/setup.bash
```
This is where all your robot code will live.
---
## 8. Save the environment as a Docker image
Exit the container:
```
exit
```
On your host:
```
docker commit {container_name} {image_name}
```
Now you can run this full ROS system anywhere:
```
docker run -it {image_name}
```
---

## 9. What you have built
You now have a:
- secure
- stable
- ROS 2 Humble
- Docker-based
- robot operating system

This is how real robots, drones, and research systems are deployed.

---

## 10. MOVING THE DOCKER IMAGE TO ANOTHER SYSTEM
After you have created the image called
```
{image_name}
```
this image contains your full ROS 2 Humble robot OS.
There are two ways to move this image to another computer.

---

# 1.OFFLINE METHOD (USB, hard drive, local copy)

On the computer where the image was created, run:
```
docker save {image_name} > {package_name}.tar
```

This creates a file called:
```
{package_name}.tar
```
This file contains the full Ubuntu + ROS system.
Copy this file to the other computer using a USB drive, external disk, or SCP.

On the other computer, load the image:
```
docker load < {package_name}.tar
```
Check that the image exists:
```
docker images
```

You should see:
```
{image_name}
```
Run it:

```
docker run -it {image_name}
```

You now have the exact same ROS system on the new machine.

---

# 2. ONLINE METHOD (Docker Hub)

This is useful if you want to share the robot OS with others.
Login to Docker Hub:

```
docker login
```
Tag your image with your Docker Hub username:

```
docker tag ros2-humble-ready yourusername/{image_name}
```

Upload the image:

```
docker push yourusername/{image_name}
```

On any other system, download it:
```
docker pull yourusername/{image_name}
```

Run it:
```
docker run -it yourusername/{image_name}
```
This gives the same ROS 2 Humble environment on any machine.

---

## How to remove the container environment from scratch

# 1. Stop and delete the ROS container
First list all containers:
```
docker ps -a
```
You will see things like:
```
{container_name}
{image_name}
```

Stop them:
```
docker stop {container_name}
docker stop {image_name}
```

Now delete them:
```
docker rm {container_name}
docker rm {image_name}
```
This removes the running Linux machines.

# 2. Delete the ROS images
List images:
```
docker images
```

You will see:
```
{image_name}
{container_name}
ubuntu
```

Remove the ROS images:

```
docker rmi {image_name}
docker rmi {container_name}
```
If Ubuntu was only used for this:
```
docker rmi ubuntu:22.04
```

# 3. Remove exported image files
If you created:
```
{package_name}.tar
```
Delete it:
```
rm {package_name}.tar
```
4. Verify everything is gone
```
docker ps -a
```
should be empty or not show ROS containers.

```
docker images
```
should not show ros2-humble or ros2-humble-ready.

---

Your system is now clean.

No ROS
No Ubuntu
No containers
No images

Like nothing ever happened.

If you ever want to rebuild it, you now know how from scratch.

---
