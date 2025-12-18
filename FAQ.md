# Frequently Anticipated Questions
Yeah, I know... It's supposed to be Frequently _Asked_ Questions. But, this is a brand new project, so for now it's _Anticipated_.

## Do I really have to build everything from source?
No. My goal is to have downloadable arm64 versions of the programs used in the instructions. In this repository you can find:

* busybox.arm64.tar.gz
* fsck.ext4.arm64.tar.gz
* fsck.fat.arm64.tar.gz

By extracting these, and using the Raspberry Pi project's [prebuilt firmware](https://github.com/raspberrypi/firmware) files, you can get through the guide without compiling.

## What about 32-bit Raspberry Pi hardware?
It should be possible, but I have no plans to include 32-bit support in this guide.

Rick Carlino's [Build a Raspberry Pi Linux System the Hard Way](https://www.rickcarlino.com/2021/build-a-raspbery-pi-linux-system-the-hard-way.html) tutorial targets ARMv7 (32-bit) and may be helpful in this respect.

## Can you incorporate _feature X_ so I can use it in my project?
Probably not. This project exists as a journey, not a destination. In other words, the instructions are here to help you get started building your own Raspberry Pi DIY OS, not provide a pre-built one for you.
