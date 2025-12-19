# Phase 4: Improving system start-up
If you're comfortable using _vi_ as an editor, you can remount the root file system as read-write and make the changes directly on the Raspberry Pi via serial console. Otherwise, you can always shutdown and move the microSD to the development VM to make the changes.

Add the file system mounts to _/etc/init.d/rcS_

```
mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
vi /etc/init.d/rcS
```

Remove the `echo "Hello World!" line and make the file contents match what's shown below.

```
mount -t proc proc /proc
mount -t sysfs sysfs /sys
```

Reboot the Raspberry Pi to test. The command is shown below.

```
reboot
```

When the Pi comes back up, take a moment to repeat the _mount_ command to verify it works and shows what file systems are currently available on the Pi. Notice root is read-only again.

## File system checks
It might be tempting to remount the root file system as read-write as part of the start-up script. But, that's not advisable until we have a way to check the file systems for potential problems first.

For this, we need `fsck.ext4` for the checking the root filesystem and `fsck.fat` for checking the boot filesystem. Busybox does not include these utilities, so we'll need to get them downloaded, built, and installed using the development VM.

First, shutdown the Pi to get the microSD transferred back to the Ubuntu VM.

```
poweroff
```

### Building fsck.ext4
The process is very similar to the way Busybox was built: clone the source code, compile for arm64 with static linking, and transfer to the microSD root file system.

Here are the commands:

```
cd ~
git clone https://github.com/tytso/e2fsprogs.git --depth=1
cd ~/e2fsprogs
./configure LDFLAGS=-static --host=aarch64-linux-gnu
make
```

One thing to notice is the files we built all have debugging info in them. This takes extra space and is not needed for our purposes. Debugging parts can be removed with the _strip_ command (or in our case, _aarch64-linux-gnu-strip_ for arm64).

```
$ file e2fsck/e2fsck
e2fsck/e2fsck: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=273a3ca94a58baabec50a44087982ce4fff37e0c, for GNU/Linux 3.7.0, with debug_info, not stripped
```

### Copying e2fsck to the microSD as fsck.ext4
Even though the binary is called e2fsck, it can check ext2, ext3, and ext4 file systems. Using a symbolic link is the traditional way to indicate it's suited for somthing other than ext2.

Assuming the microSD is mounted on the development VM, these are the commands to strip the binary, and copy it over.

```
aarch64-linux-gnu-strip ~/e2fsprogs/e2fsck/e2fsck
sudo cp ~/e2fsprogs/e2fsck/e2fsck /mnt/sbin
sudo ln -s e2fsck /mnt/sbin/fsck.ext4
```

### Building fsck.fat
The boot file system is FAT (or MS-DOS) formatted. We'll need a different _fsck_ program to check it.

The build process is nearly identical to building fsck.ext4, but there is one additional component (autoconf) needed on the Ubuntu development VM. We'll install that first, then clone the source code, build and install.

Here are the commands:

```
sudo apt-get install autoconf
cd ~
git clone https://github.com/dosfstools/dosfstools.git --depth=1
cd dosfstools
./autogen.sh
./configure LDFLAGS=-static --host=aarch64-linux-gnu
make
aarch64-linux-gnu-strip ~/dosfstools/src/fsck.fat
sudo cp ~/dosfstools/src/fsck.fat /mnt/sbin
```

### Manual testing
Before adding the file system checks to rcS, we'll first boot the Pi with the new utilities and try checking and mounting manually.

Be sure to properly unmount and disconnect the microSD card from the Ubuntu development VM and the development host. Then move the card to the Pi and apply power to start it up.

First, check the mounted file systems and verify root is read-only.

```
~ # mount
/dev/root on / type ext4 (ro,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,size=428504k,nr_inodes=107126,mode=755)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
```

Then, the run commands to check and mount the root file system.

```
~ # fsck.ext4 -p /dev/mmcblk0p2
root: clean, 447/1916928 files, 165487/7659648 blocks
~ # mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
[ 1302.842527] EXT4-fs (mmcblk0p2): re-mounted 8f85a0a1-14e1-47d9-a7ca-e65dced94447 r/w.
```

Finally, use a similar process for checking and mounting /dev/mmcblk0p1 on /boot

```
~ # fsck.fat -a /dev/mmcblk0p1
fsck.fat 4.2+git (2021-01-31)
/dev/mmcblk0p1: 11 files, 3185/130812 clusters
~ # mount -t vfat /dev/mmcblk0p1 /boot
```

### Configuring /etc/init.d/rcS
If the manual checking and mounting was successful, add the commands to rcS to ensure it happens when the system boots. As a safeguard, use the full path to fsck and mount. The example below shows this in detail, along with some comments, in rcS.

```
#! /bin/sh

# Mount pseudo filesystems
/bin/mount -t proc proc /proc
/bin/mount -t sysfs sysfs /sys

# Check and mount root and boot
/sbin/fsck.ext4 -p /dev/mmcblk0p2
/bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
/sbin/fsck.fat -a /dev/mmcblk0p1
/bin/mount -t vfat /dev/mmcblk0p1 /boot
```

> Note: we're using the full path to the _fsck_ and _mount_ commands. This is a good practice for startup scripts.

### Rebooting as a final test
The reboot command and resulting output is shown below.

```
~ # reboot
umount: devtmpfs busy - remounted read-only
[ 3486.524575] EXT4-fs (mmcblk0p2): re-mounted 8f85a0a1-14e1-47d9-a7ca-e65dced94447 ro.
The system is going down NOW!
```

>  Notice how the root file system was remounted read-only for us. This is a default setting of BusyBox when no _/etc/inittab_ file is found. It's fine for now, but should not be relied upon.

## Next steps
The system is bare bones, but usable. It would be better with network connectivity though. That's what we'll be working on in the next phase.

___
Reference: https://github.com/brgl/busybox/blob/master/examples/inittab
