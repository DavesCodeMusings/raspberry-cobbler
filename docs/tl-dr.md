# Raspberry Pi DIY OS Essentials
This very high level document assumes you know a lot of things and does not go into detail. If you need more step-by-step instructions, start with [here](index.md)

## Boot partition
You can get a pre-built boot loader and kernel from the Raspberry Pi firmware git repo

https://github.com/raspberrypi/firmware.git

You'll need: bootcode.bin, fixup.dat, start.elf, and kernel8.img. Plus the dtb overlay for your model of Pi.

You'll also need to create config.txt and cmdline.txt.

## Root partition
Busybox provides the most utility for the size. A pre-built arm64 binary is available from [this project's repo](https://github.com/DavesCodeMusings/raspberry-cobbler)

You'll also need fsck.ext4 and fsck.fat if you want to check and repair filesystems. These are pre-built as well.

## Wired networking
The driver for the Ethernet adapter is built into the kernel from the Raspberry Pi firmware repo. Busybox provides ifup and ifdown. You need to create /etc/network/interfaces and write any of the if-up.d scripts.

The Ethernet adapter (on Pi 3B at least) if USB attached, so mdev is useful in dealing with the hotplug nature of this device. Create an mdev rule that calls ifup.

## Network services
If you're using mdev to run ifup, it's best to bring up any network listeners with a script in /etc/network/if-up.d/. This will ensure the network is configured before the services try to start (as they would if you stuck them in a start-up script like /etc/init.d/rcS).

Dropbear is available pre-built. An sftp-server add-on is available as well. Other services like ntpd, httpd, ftpd, etc. are part of BusyBox.

## User accounts
Users, groups, and passwords are similar to any other Linux system, except shadow passwords are not available and the default password encryption is old and super vulnerable to cracking.

Using non-root logins will require some customizing of permissions, particularly with mdev rules to make device nodes in /dev accessible to non-root.

## Logging and other frequent file writes
To save wear on the microSD, consider putting directories like /var/log and /tmp on a RAM-based tmpfs file system. But, be aware the directory contents will not survive a reboot.

## Wireless and other hardware
Any hardware beyond what's pre-compiled into the kernel will need a kernel module and possibly firmware files.

Pre-built kernel modules are available from the same git repo the kernel came from: https://github.com/raspberrypi/firmware.git

Firmware is from the Linux kernel git repo: https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git

Helpful wifi packages wpa_supplicant and iw are available as arm64 binaries on this repo.

## Terminfo and locale data
The best place to find this is from another Linux system. Look at the tmux-side-quest document here to get details.
