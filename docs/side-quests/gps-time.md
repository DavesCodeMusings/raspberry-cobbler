# GPS Time Side Quest

**_This is a work in progress and may be incomplete or inaccurate_**

If your Pi is isolated from the internet, syncing time with Network Time Protocol (NTP) will not work. One workaround is to use a real-time clock module as described in the [I2C Realtime-Clock Side Quest](i2c-rtc.md). Another option is to use a GPS module, because GPS systems keep highly accurate time.

Adding a GPS clock is more involved than using an RTC module, but it's more accurate and not prone to drift over time. The added complexity comes from needing a utility to communicate with the GPS module and coordinating that information with the utility that provides time. In this side-quest, we'll use _gpsd_ and _chrony_, both of which are built as arm64 packages by GitHub workflows.

Adding to the compexity are runtime dependencies on _glibc_ and _ncurses_. See the [tmux side-quest](side-quests/tmux.md) and the [bash side-quest](side-quests/bash.md) for help with this.

## Attaching a GPS module

```
~# cat -n /etc/mdev.conf
     1  # device        uid:gid perms   action
     2  console         0:0     600
     3  null            0:0     666
     4  zero            0:0     666
     5  random          0:0     444
     6  urandom         0:0     444
     7  ptmx            0:0     666
     8
     9  eth[0-9]+       0:0     0       @/sbin/ifup $MDEV
    10  ttyACM0         0:0     644     @/usr/sbin/gpsd -n /dev/ttyACM0
    11
    12  $MODALIAS=.* 0:0 660 @modprobe "$MODALIAS"
```

The new lines are 10 and 12.

Line 12 is a general rule to automatically load a kernel module when a device is hotplugged. In this case, it loads the _cdc_acm_ and _usbserial_ modules when the ublox 7 GPS module is plugged in via USB.

Line 10 is used to start the _gpsd_ service as soon as _/dev/ttyACM0_ appears.

The flow goes like this:
1. The GPS module is plugged into USB, triggering an event for _mdev_.
2. _mdev_ runs `modprobe` to load the driver.
3. Loading the driver causes _/dev/ttyACM0_ to be created.
4. The appearance of _/dev/ttyACM0_ causes _mdev_ to start _gpsd_.

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

## Installing chronny
