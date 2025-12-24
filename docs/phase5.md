# Phase 5: Configuring Basic Networking
BusyBox includes commands for _ifup_ and _ifdown_, the same network admin tools available on other popular Linux distributions. We can use these commands and their associated configuration files to control our system's network interfaces.

We can start by skimming the [manual page for the configuration file /etc/network/interfaces](https://manpages.ubuntu.com/manpages/noble/man5/interfaces.5.html).

Next, we'll want to create the _interfaces_ file.

```
mkdir /etc/network
touch /etc/network/interfaces
```

## Configuring loopback
Once we have the _interfaces_ file created, we can edit it to configure the loopback address. It should look like what's shown below when we're done.

```
~ # cat /etc/network/interfaces
auto lo
iface lo inet loopback
```

Now, we can bring the loopback interface up by running `ifup` with the interface name.

```
ifup lo
run-parts: /etc/network/if-pre-up.d: No such file or directory
run-parts: /etc/network/if-up.d: No such file or directory
```

Notice there were some errors reported by the command, but the loopback is configured and up as shown by the command below.

```
~ # ip address show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

## Creating ifup/down script directories
We can fix the previous errors by creating additional directories used by _ifup_ and _ifdown_

```
mkdir /etc/network/if-pre-up.d
mkdir /etc/network/if-up.d
mkdir /etc/network/if-post-up.d
mkdir /etc/network/if-pre-down.d
mkdir /etc/network/if-down.d
mkdir /etc/network/if-post-down.d
```

## Bringing loopback up at system start
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
     7  # Check and mount root and boot
     8  /sbin/fsck.ext4 -p /dev/mmcblk0p2
     9  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    10  /sbin/fsck.fat -a /dev/mmcblk0p1
    11  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    12
    13  # Bring up loopback interface
    14  /sbin/ifup lo
```

A restart the Pi will ensure everything is working as expected.

> Spoiler: this isn't going to work as expected.

## Dealing with stale state data
When the system comes back, checking the loopback interface shows it's not configured.

```
~ # ip address show lo
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

It should have an address of 127.0.0.1/8 and it does not.

Let's try to bring it up manually.

```
~ # ip address show lo
~ # /sbin/ifup lo
ifup: interface lo already configured
```

It's not configured, but it thinks it's configured.

The culprit is _/run/ifstate_, a file that holds the state of the network interfaces. Run `cat /run/ifstate` and you'll see it shows the _lo_ interface is configured. That's because it has stale information from before the last reboot.

We can fix this by cleaning up /run after the system starts, but before the interface is brought up.

Or, better yet, we can make /run clean itself up. And this can be done by mounting a temporary file system for /run. It's similar to the way we mounted /proc and /sys earlier.

But to complicate things, there are files already in /run (ifstate and utmp).

We can use the following steps to take care of things:

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

## Verifying loopback configuration after restart
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

## Configuring Ethernet
Now that the loopback is all set up, Ethernet is next. Loopback was easy. Ethernet is more challenging because of the Raspberry Pi 3's Ethernet being attached via USB. This results in it showing up after the _rcS_ start-up script has already run. But, we'll deal with that when we get there.

For now, we'll configure an Ethernet adapter that can be brought up manually.

First, edit _/etc/network/interfaces_ and add a static IP address configuration for eth0. What's shown below is an example of a typical home network configuration. You'll want to adjust the address, netmask, and gateway parameter values for your setup. (Or don't plug the Pi's RJ45 jack into your network switch yet and it won't matter.)

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
```

Next, bring up the interface and check its status. This is similar to the way we brought up the loopback previously.

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

### Dealing with USB attached Ethernet

At this point, you may be tempted to add `ifup eth0` to _rcS_ and call it done. But let me share another spoiler...it won't work. You'll see an error in the boot messages, like the one shown below.

```
ip: can't find device 'eth0'
ip: SIOCGIFFLAGS: No such device
ip: can't find device 'eth0'
```

This happens because the Pi 3 Ethernet adapter is connected via the USB bus and the device hasn't been detected and configured by the time we're trying to configure it. We have to wait until eth0 is available. But how?

A _sleep 1 (or 2 or 3)_ might work, but it's clumsy. Another method is to use an mdev action to call _ifup_ when the interface is detected. This can be done by configuring the Pi to respond to hotplug events.

## Using mdev as a hotplug device manager
BusyBox includes the _mdev_ utility for handling hotplug devices. And there's a really good article over at codelucky.com that details how to configure it.

In brief, we'll be creating a configuration file that instructs _mdev_ to bring up our Ethernet interface when it's detected, and telling the kernel to use _mdev_ as it's hotplug device manager.

### Configuring eth0 for hotplug
Here it is:

```
~ # cat /etc/mdev.conf
# device        uid:gid perms   action
eth[0-9]+       0:0     0       @/sbin/ifup $MDEV
```

The _eth[0-9]+_ is a regular expression that will match any device like _eth0_, _eth1_, etc. This match is what triggers the rule. The _0:0_ is the user id and group id, and the next _0_ is the permissions (in octal.) Neither of these things matter, because _eth0_ is not a device node under /dev. What does matter is the _@/sbin/ifup $MDEV_ action.

mdev's actions are what let up run the _ifup_ command when the Ethernet interface is detected by the kernel. The _$MDEV_ used at the end is just the device name that caused the rule to match. In this case, _$MDEV = "eth0"_, so our action command becomes: _/sbin/ifup eth0_, just what we need to configure and bring up the interface.

> Using the regular expression _eth[0-9]+_ for the rule ensures any hotplugged Ethernet adapters can be configured and brought up. So adding a USB to Ethernet dongle later won't result in extra time spent troubleshooting.

### Running mdev hotplug at boot
To ensure things happen every time the Pi starts, we'll add a line (_echo "/sbin/mdev" > /proc/sys/kernel/hotplug_) to our _rcS_ file. We'll slip it in just before the loopback interface is configured (lines 17 and 18.)

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
    13  /bin/mount -t ext4 -o remount,rw,noatime /dev/mmcblk0p2 /
    14  /sbin/fsck.fat -a /dev/mmcblk0p1
    15  /bin/mount -t vfat -o noatime /dev/mmcblk0p1 /boot
    16
    17  # Start device manager
    18  echo "/sbin/mdev" > /proc/sys/kernel/hotplug
    19  mdev -s
    20
    21  # Bring up loopback interface
    22  /sbin/ifup lo
```

### Restarting to test
Reboot the Pi one more time just to make sure everything comes up as expected.

```
~ # ifconfig
eth0      Link encap:Ethernet  HWaddr B8:27:EB:F5:95:91
          inet addr:192.168.1.100  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:384 (384.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

## Phase 5 review
We now have a Pi wired Ethernet networking configured and working. If you used an IP that's valid and have the Pi plugged into a switch on your network, you should be able to ping it. And, we did all of it with hotplugging, so the kernel takes care of things when the interfaces are detected. This will help if we want to add any more USB Ethernet devices later.

## Next steps
Sure, we can ping the Pi, but wouldn't it be better if we could connect via Secure Shell and use other network services? That's what we'll be working on in the [next phase](phase6.md). Or, as Iron Maiden so aptly put it in Number of the Beast... [_Six-six-six_](phase6.md)

___

References:
* https://manpages.ubuntu.com/manpages/noble/man5/interfaces.5.html
* https://manpages.ubuntu.com/manpages/noble/man5/ifstate.5.html
* https://codelucky.com/mdev-command-linux/
