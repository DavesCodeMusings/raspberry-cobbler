# Raspberry Cobbler
_A Raspberry Pi DIY OS Guide_

If you've ever wanted to build a Raspberry Pi project that doesn't start with the words, "Install the latest Raspberry Pi OS image to an SD card," this is the guide for you. This is not about installing alternative distros on your Pi. This is about building your own distro from the ground up.

The guide will cover the steps for:
1. Installing the pre-built Raspberry Pi boot loader and kernel onto a microSD boot partition.
2. Configuring a development host for compiling 64-bit ARM using an Ubuntu Linux virtual machine.
3. Building a statically linked Busybox application to provide system commands and utilities.
4. Creating the root file system on the remaining microSD partition and installing Busybox to it.
5. Attaching the Pi to wired Ethernet and running services like SSH (shell), NTP (time), and HTTP (web).

## Prerequisites
The steps in the guide are not for the feint of heart. I assume you know a few things about Rasberry Pis, installing and running virtual machines, cloning from GitHub, using USB-to-serial devices, connecting GPIO pins, running commands on the Linux command-line, and so on. If you've done any programming with microcontrollers (Arduino or ESP32) and you're not afraid of _vi_, you'll probably have the right tools and skills.

Here's what you'll need (and what I'm using in the examples):
1. An internet connected Ubuntu Linux host (VirtualBox virtual machine with Ubuntu 20.4 LTS Noble.)
2. A 64-bit Raspberry Pi (model 3B.)
3. A microSD card (32G SanDisk class 10.)
4. A USB to TTL serial cable (Adafruit 954.)

## Ready to get started?
As Matt Bellamy of Muse said right before launching into Starlight at the Tokyo Zepp concert in 2013...

_[Let's do it!](https://davescodemusings.github.io/raspberry-cobbler/)_
