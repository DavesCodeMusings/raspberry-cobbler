# Phase 5: Configuring Networking
BusyBox includes commands for _ifup_ and _ifdown_, the same network admin tools available on other popular Linux distributions. We can use these commands and their associated configuration files to configure and control our system's network interfaces.

We can start by skimming the [manual page for the configuration file /etc/network/interfaces](https://manpages.ubuntu.com/manpages/noble/man5/interfaces.5.html).

Next, we'll want to create the _interfaces_ file.

```
mkdir /etc/network
touch /etc/network/interfaces
```

## Loopback
Once we have the _interfaces_ file created, we can edit it to configure the loopback address. It should look like what's shown below when we're done.

```
~ # cat /etc/network/interfaces
auto lo
iface lo inet loopback
```

Now, we can bring the loopback interface up.

```
ifup lo
run-parts: /etc/network/if-pre-up.d: No such file or directory
run-parts: /etc/network/if-up.d: No such file or directory
```

There were some errors reported by the command, but the loopback is configured and up as shown by the command below.

```
~ # ip address show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

We can fix the errors by creating additional directories used by _ifup_ and _ifdown_

```
mkdir /etc/network/if-pre-up.d
mkdir /etc/network/if-up.d
mkdir /etc/network/if-post-up.d
mkdir /etc/network/if-pre-down.d
mkdir /etc/network/if-down.d
mkdir /etc/network/if-post-down.d
```

Finally, we can edit /etc/init.d/rcS to bring the loopback interface up when the system boots.

rcS should look like this now:

```
~ # cat -n /etc/init.d/rcS
     1  #! /bin/sh
     2
     3  # Mount pseudo filesystems
     4  /bin/mount -t proc proc /proc
     5  /bin/mount -t sysfs sysfs /sys
     6
     7  # Mount temporary filesystems
     8  /bin/mount -t tmpfs run /run
     9  /bin/mount -t tmpfs tmp /tmp
    10
    11  # Check and mount root and boot
    12  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    13  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    14  /sbin/fsck.fat -a /dev/mmcblk0p1
    15  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    16
    17  # Bring up loopback interface
    18  /sbin/ifup lo
```

A restart the Pi will ensure everything is working as expected.

> Spoiler: this isn't going to work as expected.

When the system comes back, checking the loopback interface shows it's not configured.

```
~ # ip address show lo
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

It should have an address of 127.0.0.1 and it does not.

Let's try to bring it up manually.

```
~ # ip address show lo
~ # /sbin/ifup lo
ifup: interface lo already configured
```

It's not configured, but it thinks it _is_ configured.

The culprit is _/run/ifstate_, a file that holds the state of the network interfaces. Run `cat /run/ifstate` and you'll see it thinks the _lo_ interface is configured. That's because it has stale information from before the last reboot.

We need to clean up /run when the system starts.

Or, better yet, we can make /run clean itself up. And this can be done by mounting a temporary file system for /run. It's similar to the way we mounted /proc and /sys earlier.

But, there are files already in /run (ifstate and utmp). We can use the following steps to take care of things:

1. Configure rcS to mount a tmpfs file system on /run.
2. Clean out the existing /run.
3. Restart the Pi.

Let's take a look at /etc/init.d/rcS with the new mount (on line 8):

```
~ # cat -n /etc/init.d/rcS
     1  #! /bin/sh
     2
     3  # Mount pseudo filesystems
     4  /bin/mount -t proc proc /proc
     5  /bin/mount -t sysfs sysfs /sys
     6
     7  # Mount temporary filesystems
     8  /bin/mount -t tmpfs run /run
     9
    10  # Check and mount root and boot
    11  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    12  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    13  /sbin/fsck.fat -a /dev/mmcblk0p1
    14  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    15
    16  # Bring up loopback interface
    17  /sbin/ifup lo
```

When the system's back up, we can check for success by examining the file system and interface status after the restart.

```
~ # df -h /run
Filesystem                Size      Used Available Use% Mounted on
run                     453.1M      4.0K    453.1M   0% /run
~ # ip address show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

## Ethernet

Edit /etc/network/interfaces and add a static IP address configuration for eth0

What's shown below is an example of a typical home network configuration, but you'll want to adjust the address, mask, and gateway for your setup. (Or don't plug the Pi's RJ45 jack to your network switch yet and it won't matter.)

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
```

Bring up the interface and check its status

```
~ # ifup eth0
[ 1511.273779] smsc95xx 1-1.1:1.0 eth0: hardware isn't capable of remote wakeup
[ 1511.283022] smsc95xx 1-1.1:1.0 eth0: Link is Down
~ # ip address show eth0
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether b8:27:eb:f5:95:91 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 scope global eth0
       valid_lft forever preferred_lft forever
```

> The _Link is Down_ and _NO-CARRIER_ messages are because I've unplugged the network cable.

Dealing with USB attached Ethernet

At this point, you may be tempted to add a line in rcS for `ifup eth0`, but I will tell you now it won't work. You'll see an error in the boot messages, like the one shown below.

```
ip: can't find device 'eth0'
ip: SIOCGIFFLAGS: No such device
ip: can't find device 'eth0'
```

This happens because the Pi 3 Ethernet adapter is connected via the USB bus and the device hasn't been detected and configured by the time we're trying to configure it. We have to wait until eth0 is available. But how?

A _sleep 1_ might work, but it's clumsy. Another method is to use an mdev action to call _ifup_ when the interface is detected. This is what we'll do in the next phase.



References:
* https://manpages.ubuntu.com/manpages/noble/man5/interfaces.5.html
* https://manpages.ubuntu.com/manpages/noble/man5/ifstate.5.html
