# TL;DR Raspberry Pi DIY OS Essentials
This very high level document assumes you know a lot of things and does not go into detail. If you need more step-by-step instructions, start at the beginning of the project with [phase 0](phase0.md).

## Boot partition
You can get a pre-built boot loader and kernel by cloning the Raspberry Pi firmware git repo:

https://github.com/raspberrypi/firmware.git

You will need: _bootcode.bin_, _fixup.dat_, _start.elf_, and _kernel8.img_. Plus the dtb overlay for your model of Pi.

You'll also need to create your own _config.txt_ and _cmdline.txt_.

## Root partition
Busybox provides the most utility for the size. A pre-built arm64 binary is available from [this project's repo](https://github.com/DavesCodeMusings/raspberry-cobbler).

You'll also need _fsck.ext4_ and _fsck.fat_ if you want to check and repair filesystems. These are pre-built as well.

## Wired networking
The driver for the Ethernet adapter is built into the kernel from the Raspberry Pi firmware repo. Busybox provides _ifup_ and _ifdown_. You need to create your own _/etc/network/interfaces_ and write any of the _if-up.d_ scripts.

The Ethernet adapter (on Pi 3B at least) if USB attached, so mdev is useful in dealing with the hotplug nature of this device. Create an _mdev_ rule that calls _ifup_.

## Network services
If you're using _mdev_ to run _ifup_, it's best to bring up any network listeners with a script in _/etc/network/if-up.d/_. This will ensure the network is configured before the services try to start (as they would if you stuck them in a start-up script like _/etc/init.d/rcS_).

Dropbear is available as a pre-built arm64 binary. An _sftp-server_ add-on is available as well. Other services like _ntpd_, _httpd_, _ftpd_, etc. are part of BusyBox.

## User accounts
Users, groups, and passwords are similar to any other Linux system, except shadow passwords are not available and the default password encryption is old and super vulnerable to cracking.

Using non-root logins will require some customizing of permissions, particularly with mdev rules to make device nodes in _/dev_ accessible to non-root.

## Logging and other frequent file writes
To save wear on the microSD, consider putting directories like /var/log and /tmp on a RAM-based tmpfs file system. But, be aware the directory contents will not survive a reboot.

## Wireless and other hardware
Any hardware beyond what's pre-compiled into the kernel will need a kernel module and possibly firmware files.

Pre-built kernel modules are available from the same git repo the kernel came from: https://github.com/raspberrypi/firmware.git

Device firmware is from the Linux kernel git repo: https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git

Helpful wifi packages wpa_supplicant and iw are available as arm64 binaries on this repo.

## Shared librariy dependencies
All of the core binaries are statically-linked. Some of the pre-built add-on packages (available here in the workflow artifacts) will require _glibc_ or _ncurses_ as dependencies. The [bash side quest](side-quests/bash.md) has details on getting those installed.

## Terminfo and locale data
Look at the [tmux side quest](side-quests/tmux.md) to get details on what files go where and how to generate locales.
