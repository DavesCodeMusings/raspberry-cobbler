# Phase 1: Getting the Raspberry Pi to boot
Now that the development VM is set up, we can turn our attention to the original task of configuring the Raspberry Pi. And the first step in having a custom Raspberry Pi OS is being able to boot the Pi.

In this phase, we'll get a Raspberry Pi boot loader and Linux kernel installed on the microSD. Once that's done, we'll insert the microSD in the Pi and see if everything worked. This won't result in a complete system, or even give us a shell prompt, but it will get things ready for what comes next.

All the microSD card preparation is performed on the Ubuntu development VM.

> Warning: Following these steps will destroy all data on the microSD card. For best results use a new, name-brand microSD.

## Connecting the microSD to the virtual machine
Review the instructions in the previous phase for attaching the microSD to the virtual machine and verifying it's avaialble for use by Ubuntu.

Further verification can be done with the command `sudo fdisk -l`. Look for output similar to what's shown below.

```
Disk /dev/sdb: 29.72 GiB, 31914983424 bytes, 62333952 sectors
Disk model: SD/MMC/MS PRO
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x59c31a57

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdb1        8192 62333951 62325760 29.7G  c W95 FAT32 (LBA)
```

The fdisk output above confirms /dev/sdb contains a single FAT32 formatted partition slightly smaller than the advertised capacity of 32G.

## Preparing the microSD boot media
Before transferring files to the Pi's microSD card, we'll want to partition the storage space between the boot files and the system (root) files to be installed later.

### First, partition the microSD card

1. Run `sudo cfdisk /dev/sdb`
2. Ensure /dev/sdb1 is selected and choose _Resize_
3. Enter a _New size_ of "512M"
4. Move down to the _Free space_ line and choose _New_
5. Accept the default size (e.g. 29.2G)
6. Choose a _primary_ partition.
7. Select _Write_ and type "yes" to confirm.
8. Quit cfdisk.

> You may see some additional _Free space_ at the beginning of the microSD. This is used to align partitions and can be ignored. The free space we're interested in is the larger 29G atthe end.

### Then, write filesystems on the new partitions
The first small partition (512M) will be FAT32. This will hold the boot files. The second, larger partition will be formatted as ext4 to use later as the root file system.

These are the commands to get it done:

```
sudo mkfs.vfat -F32 /dev/sdb1
sudo fatlabel /dev/sdb1 BOOT
sudo mkfs.ext4 /dev/sdb2
sudo e2label /dev/sdb2 root
```

> Note: These file systems will appear on /dev/sdb1 (boot) and /dev/sdb2 (root) when mounted on the development VM via a USB attached microSD card adapter. But later, when the microSD card is used to boot the Raspberry Pi, they will appear as /dev/mmcblock0p1 (boot) and /dev/mmcblock0p2 (root).

## Fetching the files needed for booting the Pi
The Raspberry Pi project maintains regularly updated files for their boot loader and the Linux kernel. We can use these instead of building our own from source code. All we have to do is clone the repository contents to the development VM.

The commands below will copy the files down and show what's available for booting.

```
cd ~
git clone --depth=1 https://github.com/raspberrypi/firmware.git
cd ~/firmware/boot
ls
```

## Copying the basic bootloader files to the microSD card
Now that we have the files from the Raspberry Pi GitHub repository, we can select the ones we need to make our microSD bootable.

### First, mount the microSD boot partition

```
sudo mount /dev/sdb1 /mnt
```

### Then, copy the needed firmware files
The files are:
* bootcode.bin
* fixup.dat
* start.elf
* kernel8.img

Copy them like this:

```
cd ~/firmware/boot
sudo cp bootcode.bin fixup.dat start.elf kernel8.img /mnt
```

> Note: We're using _kernel8.img_ because the Pi 3B is a 64-bit machine. The "8" is a reference to armv8, the 64-bit CPU architecture. This is often used interchangably with the terms "arm64" and "aarch64".

## Creating config.txt
The Raspberry pi does not have a BIOS or UEFI setup utility. Its options are configured in a plain text file (called _config.txt_) that is read by the bootloader.
1. Create a new text file named _/mnt/config.txt_ using your favorite editor.
2. Add the following lines to _config.txt_ for a 64-bit kernel on a Raspberry Pi 3B using a serial console at 115200 bps:

```
arm_64bit=1
device_tree=bcm2710-rpi-3-b.dtb
dtoverlay=disable-bt
enable_uart=1
uart_2ndstage=1
init_uart_baud=115200
```

> If you're using a newer model Pi, adjust the device_tree entry to match your system.

## Copying the device tree file for your Pi to the microSD card
The example above references the device_tree file _bcm2710-rpi-3-b.dtb_ for Raspberry Pi 3B. If you're using a newer model Pi, you'll need to copy the file you specified in _config.txt_ 

1. In the Raspberry Pi repository's _firmware/boot_ folder, find the file that matches the name specified by the _device_tree_ parameter in config.txt
2. Copy the file to the microSD card.

For example, with the Raspberry Pi 3B:

```
sudo cp ~/firmware/boot/bcm2710-rpi-3-b.dtb /mnt
```

## Creating cmdline.txt
While _config.txt_ is used to specify hardware configuration options, another file, _cmdline.txt_, is used to specify options specific to the kernel. For example, we used _config.txt_ to enable the serial UART. Now we need to use _cmdline.txt_ to tell the kernel that serial line is going to be the system console.

Here's how to do it:
1. Create a new text file called _/mnt/cmdline.txt_
2. Edit the file to add the following line:

```
console=serial0,115200
```

## Copying the license information files
While this is not required for the Raspberry Pi to boot, it is required for license compliance.

1. In the firmware boot directory, find COPYING.linux and LICENCE.broadcom
2. Copy these files to the microSD card alongside the other files.

```
cd ~/firmware/boot
sudo cp COPYING.linux LICENCE.broadcom /mnt
```
## Verifying the boot files and configuration files
Before moving the microSD card to the Raspberry Pi, do one final inspection to make sure everything looks right on the boot file system.

```
$ ls -1 /mnt
bcm2710-rpi-3-b.dtb
bootcode.bin
cmdline.txt
config.txt
COPYING.linux
fixup.dat
kernel8.img
LICENCE.broadcom
start.elf
```

```
$ cat /mnt/config.txt
arm_64bit=1
device_tree=bcm2710-rpi-3-b.dtb
dtoverlay=disable-bt
enable_uart=1
uart_2ndstage=1
init_uart_baud=115200
```

```
$ cat /mnt/cmdline.txt
console=serial0,115200
```

## Moving the microSD to the Raspberry Pi
Disconnect the microSD card from the development VM and eject from the development host before removing it.

1. Unmount the microSD boot partition with `sudo umount /mnt`
2. Disconnect the microSD from the Ubuntu virtual machine (_Devices > USB_ if using VirtualBox.)
3. Eject the microSD card from the development host operating system.
4. Ensure the Raspberry Pi is disconnected from power.
5. Insert the microSD card into the Pi.

## Booting the Raspberry Pi
Now comes the moment of truth. Will it boot?

1. Attach the Pi UART GPIO pins to your USB-TTL serial cable (pins 6, 8, and 10, labeled GND, GPIO14, and GPIO15.)
2. Start PuTTY (or your favorite serial terminal) with the appropriate COM port and bit rate configured. 
3. Apply power to the Pi.
4. (Hopefully) watch the boot loader and kernel start-up messages fly by.
5. Realize the final kernel panic message is normal since we have not bulit a root file system yet.

It should start with a messages from the bootloader, transition to messages from the Linux kernel, then stop because it can't find a root file system to mount.

Here's an example, edited for brevity:

```
Raspberry Pi Bootcode
Read File: config.txt, 108
Read File: start.elf, 3027424 (bytes)
Read File: fixup.dat, 7367 (bytes)
...
[    0.000000] Linux version 6.12.61-v8+
...
[    3.703866] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

Notice the green LED on the Pi is not lit.

## Phase 1 review
Though the result of _Unable to mount root fs_ is somewhat anticlimactic, we did get a fair bit accomplished. Let's look at what we learned:

* How to create partitions and file systems for the Raspberry Pi microSD card.
* Which files are required for the boot loader and kernel when booting the Raspberry Pi.
* How to view boot and kernel messages using the serial UART.

## Next steps
Now that we can boot, we'll concentrate on getting a root file system for the kernel to mount. But first, we need to get Ubuntu ready for building binaries that will run on the Raspberry Pi 3's 64-bit arm CPU.

To misquote Nancy Sinatra...

_These boots are made for walkin'. And that's just what they'll do. One of these days these boots are gonna walk into [phase 2](phase2.md)_

___

References:
* https://forums.raspberrypi.com/viewtopic.php?t=11258
* https://raspberrypi.stackexchange.com/questions/10442/what-is-the-boot-sequence
* https://www.raspberrypi.com/documentation/computers/config_txt.html
* https://nayab.xyz/rpi3b-elinux/embedded-linux-rpi3-030-boot-process.html
* https://www.raspberrypi.com/documentation/computers/linux_kernel.html
* https://www.rickcarlino.com/2021/build-a-raspbery-pi-linux-system-the-hard-way.html
* https://www.raspberrypi.com/documentation/computers/config_txt.html#enable_uart
