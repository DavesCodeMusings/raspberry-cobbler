# Frequently Anticipated Questions
Yeah, I know... It's supposed to be Frequently _Asked_ Questions. But, this is a brand new project, so for now it's _Anticipated_.

## Do I really have to build everything from source?
No. I have downloadable arm64 versions of the programs used in the instructions. In this repository you can find:

* busybox.arm64.tar.gz
* dropbear.arm64.tar.gz
* fsck.ext4.arm64.tar.gz
* fsck.fat.arm64.tar.gz
* openssh-sftp-server.arm64.tar.gz
* tmux.arm64.tar.gz

By extracting these, and using the Raspberry Pi project's [prebuilt firmware](https://github.com/raspberrypi/firmware) files, you can get through the guide without compiling.

## What about older 32-bit Raspberry Pi hardware?
It should be possible, but I have no plans to include 32-bit support in this guide.

Rick Carlino's [Build a Raspberry Pi Linux System the Hard Way](https://www.rickcarlino.com/2021/build-a-raspbery-pi-linux-system-the-hard-way.html) tutorial targets ARMv7 (32-bit) and may be helpful in this respect.

## Can you incorporate _feature X_ so I can use it in my project?
Probably not. This project exists as a journey, not a destination. In other words, the instructions are here to help you get started building your own Raspberry Pi DIY OS, not provide a pre-built one for you.

## Why not use musl libc or ulibc to save space?
One of the goals of this project is to be accessible to people getting started with embedded Linux. Cross-compiling for arm CPU architecture can be challenging enough. Adding a different C library on top would only increase the complexity further. Static linking everything with glibc only consumes about 40M of space. Even on an 8G microSD card, this is a trivial amount of storage.

# What about Samba for DIY Network Attached Storage (NAS)?
If you can compile statically-linked Samba using an arm64 Ubuntu machine, let me know how you did it. I will create a GitHub action to automate the build.
