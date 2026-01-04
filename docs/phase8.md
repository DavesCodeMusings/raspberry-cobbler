# Setting Up System Logging
The utilities we need for logging (_/syslogd_ and _klogd_) were already installed as part of the BusyBox package. What we'll concentrate on here is how to enable logging while minimizing wear on the microSD card.

## Understanding the challenges of SD cards as storage
The microSD cards used by the Raspberry Pi don't last forever. Their shortcoming is they lack the wear-leveling features of the more robust Solid-State Drives (SSDs.) Constant writing to a microSD can reduce its life and eventually cause it to fail.

When we think about logging, we need to consider the impact of constantly writing logs the microSD card.

One way around this is to use the RAM-based _tmpfs_ file system like we used for _/tmp_ and _/run_. The drawback is we'll lose the logs whenever the Pi reboots. It's a trade-off, data loss for microSD life.

## Creating the log directory
The traditional place for Linux systems to store logs is _/var/log/messages_. We'll create the log directory with a _tmpfs_ mount.

```
mkdir /var/log
/bin/mount -t tmpfs log /var/log
```

Check that everything looks good using the `mount` command with no parameters. Anong the other file systems, you should see /var/log.

```
~ # mount
log on /var/log type tmpfs (rw,relatime)
```

To make sure this happens every time the system starts, add the lines to _/etc/init.d/rcS_ just after the lines that mount _/run_ and _/tmp_.

```
~ # cat -n /etc/init.d/rcS
     1  #! /bin/sh
     2
     3  # Mount pseudo filesystems
     4  /bin/mount -t proc proc /proc
     5  /bin/mount -t sysfs sysfs /sys
     6  /bin/mkdir /dev/pts
     7  /bin/mount -t devpts devpts /dev/pts
     8
     9  # Mount temporary filesystems
    10  /bin/mount -t tmpfs run /run
    11  /bin/mount -t tmpfs tmp /tmp
    12  /bin/mount -t tmpfs log /var/log
    13
    14  # Check and mount root and boot
    15  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    16  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    17  /sbin/fsck.fat -a /dev/mmcblk0p1
    18  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    19
    20  # Start device manager
    21  echo "/sbin/mdev" > /proc/sys/kernel/hotplug
    22  mdev -s
    23
    24  # Bring up loopback interface
    25  /sbin/ifup lo
```

Line 12 shows mounting a RAM-based file system on /var/log.

> If you decide permanent log storage is more important than microSD lifespan, simply skip the step for mounting a tmpfs filesystem on /var/log.

## Starting syslogd and klogd
Two system programs need to be run. First, _syslogd_ to start the system logging service. Then, _klogd_ to send the kernel logs to the system log service.

You can start them manually now, or just add them to _rcS_ and restart the system.

```
~ # cat -n /etc/init.d/rcS
     1  #! /bin/sh
     2
     3  # Mount pseudo filesystems
     4  /bin/mount -t proc proc /proc
     5  /bin/mount -t sysfs sysfs /sys
     6  /bin/mkdir /dev/pts
     7  /bin/mount -t devpts devpts /dev/pts
     8
     9  # Mount temporary filesystems
    10  /bin/mount -t tmpfs run /run
    11  /bin/mount -t tmpfs tmp /tmp
    12  /bin/mount -t tmpfs log /var/log
    13
    14  # Start logging
    15  /sbin/start-stop-daemon -S -x /sbin/syslogd
    16  /sbin/start-stop-daemon -S -x /sbin/klogd
    17
    18  # Check and mount root and boot
    19  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    20  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    21  /sbin/fsck.fat -a /dev/mmcblk0p1
    22  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    23
    24  # Start device manager
    25  echo "/sbin/mdev" > /proc/sys/kernel/hotplug
    26  mdev -s
    27
    28  # Bring up loopback interface
    29  /sbin/ifup lo
```

Lines 15 and 16 show syslogd and klogd being started.

## Verifying setup
Whether you started the services manually or if you rebooted, the effect should be the same. /var/log should be mounted, _syslogd_ and _klogd_ should be running, and there should be a _/var/log/messages_ file.

```
~ # mount | grep log
log on /var/log type tmpfs (rw,relatime)
~ # ps -ef | grep logd
  108 root      0:00 /sbin/syslogd
  110 root      0:00 /sbin/klogd
  647 root      0:00 grep logd
~ # ls -lh /var/log/messages
-rw-r--r--    1 root     root       30.2K Dec 23  2025 /var/log/messages
```

## Reading logs
We can `cat` or `tail` /var/log/messages to see what the system wants to tell us.

Do that right after boot, and you'll see messages from the kernel along with an entry about starting /bin/sh on the console (tty).

Remember /var/log is RAM-based, so everything gets wiped out with a reboot of the system.

## Quieting the console
We can calm the number of messages sent to the console by using the file _/proc/sys/kernel/printk_

Reading from the file will tell us the current configuration.

```
~ # cat /proc/sys/kernel/printk
7       4       1       7
```

The first number is the current level. 7 is debug, which means a lot of messages will flood the console.

Writing to the file will set the desired log level.

```
~ # echo 3 > /proc/sys/kernel/printk
~ # cat /proc/sys/kernel/printk
3       4       1       7
```

The effect is immediate.

We can use this in _rcS_, right after _klogd_ is started, to send most messages to _/var/log/messsages_ rather than to the console.

## Phase 8 review
Nearly everything we did in this phase parallels something we've done before: mounting a tmpfs file system, starting system services, and using _rcS_ to handle things at start-up. A few short commands and now we can get log information from _/var/log/messages_.

## Next Steps
As we moved through the phases of the project, the focus has been on trying things and fixing what doesn't work. Because of this approach, we've got some gaps in the system. This is what we'll look at in the [next and final phase](phase9.md).
