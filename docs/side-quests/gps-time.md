# GPS Time Side Quest

**_This is a work in progress and may be incomplete or inaccurate_**

If your Pi is isolated from the internet, syncing time with Network Time Protocol (NTP) will not work. One workaround is to use a real-time clock module as described in the [I2C Realtime-Clock Side Quest](i2c-rtc.md). Another option is to use a GPS module, because GPS systems keep highly accurate time.

Adding a GPS clock is more involved than using an RTC module, but it's more accurate and not prone to drift over time. The added complexity comes from needing a utility to communicate with the GPS module and coordinating that information with the utility that provides time. In this side-quest, we'll use _gpsd_ and _chrony_, both of which are built as arm64 packages by GitHub workflows.

Adding to the compexity are runtime dependencies on _glibc_ and _ncurses_. See the [tmux side-quest](side-quests/tmux.md) and the [bash side-quest](side-quests/bash.md) for help with this.

## Attaching a GPS device
The GPS device in this example is a ublox 7 device plugged into USB. It needs a kernel module to work. The module needs to be insalled when the device is plugged in. For this hotplugging task, we turn to _mdev_

```
~ # cat -n /etc/mdev.conf
     1  # device        uid:gid perms   action
     2  console         0:0     600
     3  null            0:0     666
     4  zero            0:0     666
     5  random          0:0     444
     6  urandom         0:0     444
     7  ptmx            0:0     666
     8
     9  eth[0-9]+       0:0     0       @/sbin/ifup $MDEV
    10
    11  $MODALIAS=.* 0:0 660 @modprobe "$MODALIAS"
```

The new line is appended at the end.

Line 11 is a general rule to automatically load a kernel module when a device is hotplugged. In this case, it loads the _cdc_acm_ and _usbserial_ modules when the ublox 7 GPS module is plugged in via USB. This will cause the creation of a new device, _/dev/ttyACM0_.

## Communicating with the GPS module
To test...

```
#! /bin/sh

# Display data from serial attached GPS device for testing.

PORT=/dev/ttyACM0
SPEED=9600

echo "Reading from $PORT at $SPEED bps."
stty -F $PORT $SPEED

while read LINE < $PORT; do
    echo $LINE
done
```

## Installing gpsd
Assuming the package has been downloaded to /tmp.

```
cd /tmp
unzip gpsd.zip
cd /
tar -xf /tmp/gpsd.arm64.tar
rm /tmp/gpsd.*
```

Starting _gpsd_ is not as simple as adding the command to _rcS_. It won't start if the serial port for the GPS device is not available.

```
~ # cat -n /etc/mdev.conf
     1  # device        uid:gid perms   action
     2  console         0:0     600
     3  null            0:0     666
     4  zero            0:0     666
     5  random          0:0     444
     6  urandom         0:0     444
     7  ptmx            0:0     666
     8
     9  eth[0-9]+       0:0     0       @/sbin/ifup $MDEV
    10  ttyACM0         0:0     644     @/usr/sbin/gpsd --nowait /dev/ttyACM0
    11
    12  $MODALIAS=.* 0:0 660 @modprobe "$MODALIAS"

```

Line 10 is used to start the _gpsd_ service as soon as _/dev/ttyACM0_ appears. It works in conjuction with the _modprobe_ action on line 12.

The flow goes like this:
1. The GPS module is plugged into USB, triggering an event for _mdev_.
2. Line 12 tell _mdev_ to run `modprobe` to load the driver.
3. Loading the driver causes _/dev/ttyACM0_ to be created.
4. On line 10, the appearance of _/dev/ttyACM0_ causes _mdev_ to start _gpsd_.

## Testing GPS fix on satellites

```
~ # cgps
```

If all goes well, the result will be a list of the satellites the GPS device can see as well as how many it's using to maintain a position fix.

## Installing chronny
Assuming the package has been downloaded to /tmp.

```
cd /tmp
unzip chrony.zip
cd /
tar -xf /tmp/chrony.arm64.tar
rm /tmp/chrony.*
```

Create a basic configuration file fo chrony. The example below assumes no connection to the internet, so no upstram servers are configured, only the GPS and RTC are used as time sources.

```
~ # cat -n /etc/chrony.conf
     1  refclock SHM 0 offset 0.5 delay 0.2 refid NMEA
     2  refclock RTC /dev/rtc0:utc stratum 8
     3  rtconutc
     4
     5  driftfile /var/lib/chrony/drift
     6  logdir /var/log
     7  #log measurements statistics tracking
     8
     9  allow all
```

The last line, _allow all_, is what instructs _chrony_ to act as a time server (port 123/udp). Because _chrony_ is a server, starting has to wait until there is a network interface available. This means _chrony_ gets started by a script in /etc/network/if-up.d_.

```
# cat -n /etc/network/if-up.d/chronyd.sh
     1  #!/bin/sh
     2  if [ "$IFACE" != "lo" ]; then
     3    start-stop-daemon -S -x /usr/sbin/chronyd
     4  fi
```

> Be sure to make the script executable or you will be sad when it doesn't run.

## Testing
A reboot of the system will let us know if everything is configured correctly.

Make sure things are running.

```
~ # ps -ef | grep gpsd
  214 nobody    0:00 /usr/sbin/gpsd --nowait /dev/ttyACM0
~ # ps -ef | grep chronyd
  168 root      0:00 /usr/sbin/chronyd
```

Check _chrony's_ view of time sources.

```
~# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
#* NMEA                          0   4    77    11  +2273us[+3041us] +/-  100ms
#- RTC1                          8   4    77    10   +755ms[ +755ms] +/-  230us
```

## What doesn't work...and how to deal with it
Try the familiar `hwclock` command to check the real-time clock and you may be in for a surprise.

```
~# hwclock
hwclock: can't open '/dev/rtc0': Device or resource busy
```

It doesn't work. That's because _chrony_ has exclusive access to _/dev/rtc0_. Killing the _chronyd_ process will relinquish control, but then of course, this means the RTC is no longer getting updates from the GPS device.

The solution is to use _chronyc_ to make updates to the RTC. One such method is `chronyc makestep`. Using the _help_ command at the _chronyc_ prompt will explain what it does.
