# Phase 3: Creating a root file system with BusyBox utilities
If you wanted a full-featured Raspberry Pi, you probably wouldn't be reading this guide and would instead just flash Raspberry Pi OS on your microSD and be done. So we won't be trying to duplicate even a fraction of what's already available in Raspberry Pi OS. Our focus is on minimalism.

Taking cues from other minimalist Linux distributions, like [Alpine Linux](https://www.alpinelinux.org/) and [OpenWrt](https://openwrt.org/), we'll turn to the [BusyBox project](https://busybox.net/) for most of our operating system utilities.

## Installing required BusyBox dependencies
Before we can build BusyBox, there are a few packages we'll need on our development VM in order to successfully configure and compile. The commands to install these is shown below.

```
sudo apt-get update
sudo apt-get install libncurses-dev bzip2 libssl-dev
```

Now we're ready to build BusyBox.

## Fetching BusyBox source code
The BusyBox project page has the source code we need. We'll clone their repository onto the development VM.

```
cd ~
git clone https://git.BusyBox.net/BusyBox/ --depth=1
```

## Configure BusyBox features
In this step we'll configure BusyBox for the Raspberry Pi's CPU architecture and also disable one of the utilities that causes compiler errors with newer versions of the Linux kernel.

```
cd ~/busybox
sudo make menuconfig
```

Using default settings is mostly fine, but be sure to configure for static linking and cross-compiling as shown below.

From the _Settings_ submenu selection, scroll down to the _--- Build Options_ heading, and configure the following options as shown:
* _[*] Build static binary (no shared libs)_
* _(aarch64-linux-gnu-) Cross compiler prefix_

Still in the _Settings_ submenu, scroll further to find the _--- Installation Options_ heading and configure the _Destination path_
* _(/mnt) Destination path for 'make install'_

Exit the _Settings_ submenu and under the _Applets_ heading, find and enter the _Networking Utilities_ submenu. From there, find and disable _tc_ as a bug workaround.
* _[ ] tc (8.3 kb)_
  
> Note: You can customize aditional applets if you like, but with the exception of the _tc_ bug, the defaults should be fine.

## Building BusyBox
Run the _make_ command to compile. We'll prefix with sudo to avoid errors that occur when attempting to set permissions on files as a regular user.

```
sudo make
```

Compilation takes a while, even on a fast machine. But, you should see steady progress. Warnings in the output messages are okay, but errors will stop compilation and need to be investigated. Most times it will be a missing dependency that can be solved with an appropriate `apt search` to determine the package name and `apt-get install` to install it.

## Installing BusyBox
Again, we'll need to make the microSD available to the virtual machine before mounting. Review the previous sections to find the instructions if you need them.

Once the microSD is available to the development VM, we can mount the root filesystem and install the BusyBox utilities.

```
sudo e2fsck /dev/sdb2
sudo mount /dev/sdb2 /mnt
cd ~/busybox
sudo make install
sudo chmod u+s /mnt/bin/busybox
```

## Installing remaining directory structure
BusyBox only installs the minimum it needs. To have a more functional system, we can add a few directories while the microSD is mounted on the development VM.

```
cd /mnt
sudo mkdir boot dev etc media mnt proc run sys tmp var
sudo mkdir etc/init.d
sudo ln -s /run /mnt/var/run
sudo chmod 1777 tmp
sudo ln -s /tmp /mnt/var/tmp
```

## Creating a temporary proof of concept _rcS_
After the root file system is mounted, BusyBox will take over system startup. By default, it looks to the file _/etc/init.d/rcS_ for the commands to execute.

We can create a simple _rcS_ that just echoes "Hello World!" to show us our system is working.

```
cd /mnt/etc/init.d
sudo touch rcS
sudo chown 0:0 rcS
sudo chmod 755 rcS
sudo vi rcS
```

Add the line: `echo "Hello World!"` to _rcS_ then save.

## Edit cmdline.txt on the boot partition to add the root file system
The _cmdline.txt_ file we created earlier informed the kernal about the serial console. Now we need to tell the kernel about the root file system we want to use.

1. Mount the boot filesystem on _/mnt/boot_
2. Edit _cmdline.txt_
3. After _console=serial0,115200_, add a space and append `root=/dev/mmcblk0p2 rootfstype=ext4 rootwait`

```
you@Ubuntu:~$ sudo mount /dev/sdb1 /mnt/boot
you@Ubuntu:~$ sudo vi /mnt/boot/cmdline.txt
console=serial0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait
```

## Booting the Pi with the microSD
Review the process for detaching the microSD from the Ubuntu virtual machine and the host operating system to ensure there is no risk of data corruption.

1. Unmount any microSD file systems on the Ubuntu virtual machine (/mnt/boot and /mnt).
2. Ensure the microSD has been properly disconnected from the VM (_Devices > USB_) and ejected from the development host.
2. Transfer the microSD to the Raspberry Pi.
3. Start up PuTTY and connect to the USB-serial adapter COM port.
4. Apply power to the Pi.

Look for the _Hello World!_ message to verify rcS ran correctly, then press Enter to get a shell prompt.

> Note: _Hello World!_ may be buried in kernel messages and you might need to scroll up a bit to see it.

## Phase 3 review
Not only will you see the _Hello World!_ message, proving rcS ran, there's also a prompt to press Enter for a console. Press the button. You know you want to.

There's a shell prompt, and most of the basic commnds work. But, if you try any administrative commands, they're not very successful.

The commands below will show some examples of these shortcomings.

```
~ # date
Thu Jan  1 00:02:42 UTC 1970
~ # mount
mount: no /proc/mounts
~ # ip address show
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop qlen 1000
    link/ether b8:27:eb:f5:95:91 brd ff:ff:ff:ff:ff:ff
```

Some of this can be temporarily overcome by mounting _/proc_ and _/sys_ as shown below.

```
~ # mount -t proc proc /proc
~ # mount -t sysfs sysfs /sys
~ # mount
/dev/root on / type ext4 (ro,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,size=428504k,nr_inodes=107126,mode=755)
/dev/mmcblk0p1 on /boot type vfat (ro,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,errors=remount-ro)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
~ # ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop qlen 1000
    link/ether b8:27:eb:f5:95:91 brd ff:ff:ff:ff:ff:ff
~ # date
Thu Jan  1 00:02:44 UTC 1970
```

We can see the mounted filesystems now, but the network configuration and system time is still not right.

## Next steps
In the next phase, we should work on the start-up scripts so the file system mounting and other things can happen automatically.

In the immortal words of James Brown counting up to the break in The Funky Drummer... [1, 2, 3, 4, Hit it!](phase4.md)

___

References:
* https://busybox.net/source.html
* https://lists.busybox.net/pipermail/busybox-cvs/2024-January/041752.html
* https://www.rickcarlino.com/2021/build-a-raspbery-pi-linux-system-the-hard-way.html
