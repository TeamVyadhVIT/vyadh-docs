---
title: Install ROS 2 Humble on Ubuntu 22.04 Desktop
date: 2026-06-01
author: santosh
categories: [software, general]
tags: [ros2, humble, ubuntu]
---

# Install ROS 2 Humble on Ubuntu 22.04

This guide follows the official Humble apt install flow.

## 1. Check your locale

ROS works best with a UTF-8 locale. This is just a quick check before installing anything.

```bash
locale
```

This prints your current language and character settings.
If you already see `UTF-8` in the output, you can continue.

If you do not have a UTF-8 locale yet, run these commands:

```bash
sudo apt update
sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale
```

`sudo apt update` refreshes Ubuntu's package list.
`sudo apt install locales` installs the locale tools.
`sudo locale-gen en_US en_US.UTF-8` creates the UTF-8 locale.
`sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8` saves the new default language settings.
`export LANG=en_US.UTF-8` applies the setting in the current terminal.
`locale` shows the result so you can confirm it worked.

## 2. Enable the ROS package source

Ubuntu needs a few setup packages before it can download ROS from the official ROS repository.

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

`software-properties-common` gives Ubuntu the command for managing software repositories.
`universe` enables the Ubuntu repository that ROS packages depend on.

Now install the ROS apt source helper and register the Humble repository:

```bash
sudo apt update
sudo apt install curl -y
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F'"' '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
```

`sudo apt update` refreshes the package list again after enabling the new repository.
`sudo apt install curl -y` installs the download tool used in the next commands.
`export ROS_APT_SOURCE_VERSION=...` finds the latest ROS apt source release tag.
`curl -L -o /tmp/ros2-apt-source.deb ...` downloads the ROS repository setup package.
`sudo dpkg -i /tmp/ros2-apt-source.deb` installs the package that adds the ROS 2 repository to Ubuntu.

## 3. Install ROS 2 Humble Desktop

Refresh the package list one more time, then install the desktop bundle.

```bash
sudo apt update
sudo apt install ros-humble-desktop
sudo apt install ros-dev-tools
```

`sudo apt update` makes sure Ubuntu can see the new ROS packages.
`sudo apt install ros-humble-desktop` installs ROS 2 Humble with RViz, demos, and tutorials.
`sudo apt install ros-dev-tools` installs extra tools you will need later when building your own ROS packages.

Iinstall the ROS-Gazebo integration package after the desktop install:

```bash
sudo apt install ros-humble-gazebo-ros-pkgs
```

This adds the ROS packages that connect ROS 2 with Gazebo simulation.
If Ubuntu says the package is missing, run `sudo apt update` again and try once more.

## 4. Set up your terminal

Before you run any ROS command, source the Humble setup file in the terminal you are using.

```bash
source /opt/ros/humble/setup.bash
```

We recommend you use nano to add the above line to your `.bashrc` file. This way you wouldn't have to do it manually every time.
```
sudo nano ~/.bashrc
```

This loads the ROS 2 commands into your current terminal session.
If you close the terminal, you will need to run this command again in the new one.

## 5. Test the install

Open two terminals and run the example talker and listener.

In terminal 1:

```bash
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp talker
```

`source /opt/ros/humble/setup.bash` makes the ROS commands available.
`ros2 run demo_nodes_cpp talker` starts a demo program that keeps publishing messages.

In terminal 2:

```bash
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_py listener
```

`source /opt/ros/humble/setup.bash` loads ROS again in the second terminal.
`ros2 run demo_nodes_py listener` starts a demo program that listens for those messages.

If everything is working, the listener should print messages from the talker.
That means ROS 2, RViz support, and the command-line tools are installed correctly.

