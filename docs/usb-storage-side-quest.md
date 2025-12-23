# USB Mass Storage Side Quest
Here we'll take a salvaged SATA spinning disk and attach it via a USB-SATA adapter to see what we can do with it. Along the way we'll create a method for mounting USB disks whenever mdev detects them. The instructions use a spinning disk, but solid state should work the same. Just be careful not to drawn too much power from the Pi's USB ports.

## Figuring out how USB attached disks show up

Plugging it in gives these (abridged) messages on the console:

```
~ # [51983.417731] usb 1-1.3: new high-speed USB device number 5 using dwc_otg
[51983.607016] usb 1-1.3: New USB device found, idVendor=152d, idProduct=0578, bcdDevice= 3.01
[51983.615533] usb 1-1.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[51983.622964] usb 1-1.3: Product: USB 3.0 Device
[51983.627480] usb 1-1.3: Manufacturer: USB 3.0 Device
[51983.632437] usb 1-1.3: SerialNumber: 000000000069
[51983.678447] usb-storage 1-1.3:1.0: USB Mass Storage device detected
[51983.685214] usb-storage 1-1.3:1.0: Quirks match for vid 152d pid 0578: 1000000
[51983.692800] scsi host0: usb-storage 1-1.3:1.0
[51984.702722] scsi 0:0:0:0: Direct-Access     Hitachi  HTS543216L9A300  0301 PQ: 0 ANSI: 6
[51984.733331] sd 0:0:0:0: [sda] 312581808 512-byte logical blocks: (160 GB/149 GiB)
[51984.741462] sd 0:0:0:0: [sda] Write Protect is off
[51984.809213]  sda: sda1 sda2 sda3 sda4
[51984.814379] sd 0:0:0:0: [sda] Attached SCSI disk
```

We can see the USB-SATA adapter detected (usb 1-1.3), then the disk (usb-storage 1-1.3:1.0 and scsi 0:0:0:0), and finally the disk (sda) partitions (sda1..4).

The _sda_ device and its partitions show up in /dev without any interaction required on our part.

```
~ # ls /dev/sd*
/dev/sda   /dev/sda1  /dev/sda2  /dev/sda3  /dev/sda4
```

We can even see the partition table with _fdisk_

```
~ # fdisk -l /dev/sda
Disk /dev/sda: 149 GB, 160041885696 bytes, 312581808 sectors
19381 cylinders, 256 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device  Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/sda1    0,0,2       1023,255,63          1  312581807  312581807  149G ee EFI GPT
```

Trying to run _fsck_ for an ext4 filesystem gives an error.

```
~ # fsck.ext4 /dev/sda1
e2fsck 1.47.3 (8-Jul-2025)
ext2fs_open2: Bad magic number in super-block
fsck.ext4: Superblock invalid, trying backup blocks...
fsck.ext4: Bad magic number in super-block while trying to open /dev/sda1
...
/dev/sda1 contains a vfat file system
```

But, fsck.ext4 eventually figures out it's a VFAT (Windows) file system and tells us so.

Using _fsck.fat_ is the way to check VFAT disks.

```
~ # fsck.fat /dev/sda1
fsck.fat 4.2+git (2021-01-31)
/dev/sda1: 5 files, 135/51078 clusters
```

We can even mount it if we want to see what's on the disk.

```
~ # mount /dev/sda1 /mnt
~ # ls /mnt
```

## Repurposing the disk
If there's nothing on the disk worth keeping, we might as well re-use it for the Pi. We have a couple choices for file systems.
* FAT - Old technology and not very robust, but can be used by both Windows and Linux systems.
* ext4 - More features (like permissions and journaling), but limited to Linux only use.

### Formatting as Windows compatible
First, run `fdisk /dev/sda` to delete any partitions and create a new one of type 0x0C (Win95 FAT32 LBA).

Then, use `mkfs.vfat sda1` to create the new file system.

### Formatting as Linux compatible
First, run `fdisk /dev/sda` to delete any partitions and create a new one of type 0x83 (Linux).

Then, use `mkfs.ext4 /dev/sda1` to create the new file system.

### Adding to /etc/fstab
Assuming you chose to format as ext4, we could do something simple like this for fstab and it would work.

```
/dev/sda1  /media  ext4  defaults  0  0
```

But, we're dealing with a USB device that can be plugged or removed at will, so we really can't depend on this particular disk always being /dev/sda1. It's much better to refer to the disk partition by it's UUID. And we can get the UUID from the command `blkid /dev/sda1`.

Using the output from `blkid` we can contruct a better /etc/fstab entry using the UUID as the device name.

```
UUID=31415926-5358-9793-2384-626433832795  /media  ext4  defaults  0  0
```  

# Using mdev for hotplug detection
Since mdev is already handling our USB Ethernet adapter, getting it to react to a USB disk is trivial. We'll append a rule to call a script to mount the USB storage whenever its presence is detected.

```
~ # cat /etc/mdev.conf
# device        uid:gid perms   action
eth[0-9]+       0:0     0       @/sbin/ifup $MDEV
sd[a-z][0-9]    0:0     660     @/etc/mdev/usb-storage.sh $MDEV
```

## Creating a hotplug mount script
The script that does the mounting is a little more involved. From our experience with mounting boot and root file systems, we know we want to run fsck first. But, we'll need to do some validation checks before attempting any of this. What kind of file system is it? What's its UUID?

We can use blkid to determine the UUID and file system type so we know how to check it and mount it.

```
~ # cat -n /etc/mdev/usb-storage.sh
     1  #! /bin/sh
     2
     3  # Helper to mount an mdev detected storage device.
     4  # Use with mdev.conf rule:
     5  #   sd[a-z][0-9]    0:0     660     @/etc/mdev/usb-storage.sh $MDEV
     6  # Requires fstab entries to use UUID device names.
     7
     8  DEV=$1
     9  if ! [ -b "$DEV" ]; then
    10      echo "Block device not found: $DEV"
    11      exit 1
    12  else
    13      # Same regex, but returning different capture values.
    14      UUID=$(blkid $DEV | sed 's/.*UUID="\(.*\)" TYPE="\(.*\)"/\1/')
    15      TYPE=$(blkid $DEV | sed 's/.*UUID="\(.*\)" TYPE="\(.*\)"/\2/')
    16      if [ -z "$UUID" ] || [ -z "$TYPE" ]; then
    17          echo "Empty UUID or TYPE device returned for $DEV"
    18          exit 2
    19      else
    20          /sbin/fsck.$TYPE -p $DEV && /bin/mount UUID=$UUID
    21      fi
    22  fi
```

## Testing
Removing and re-plugging the USB disk should trigger the mdev rule and call the device mounting script. Restarting the system should accomplish the same test. Watch for messages on the console.

Running the `mount` command with no parameters will show all mounted devices.

___

Reference: https://codelucky.com/mdev-command-linux/
