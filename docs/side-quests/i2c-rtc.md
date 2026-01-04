# I2C Real-Time Clock Side Quest
If your Pi is not attached to the internet for NTP time syncing, you might get tired of constantly seeing Jan 1, 1970. Fortunately, this can be remedied with an inexpensive real-time clock (RTC) module.

The instructions here assume a DS3231 (DS1307 compatible) RTC module, attached to the I2C bus on GPIO2 (header pin 3) for data and GPIO3 (header pin 5) for clock. The module I purchased has a 5-pin socket that fits over GPIO pins 1, 3, 5, 7, and 9 for 3.3V, SDA, SCL, (unused), and GND.

There are many different device configurations out there, so be sure to check your wiring twice before powering up.

## Enabling I2C
The first step to getting the RTC to function is enabling the I2C bus it uses to communicate. This involves:

* Enabling I2C in _config.txt_
* Loading kernel modules for I2C
* Verifying the device address

The Pi's I2C bus is not enabled by default. A line needs to be added to _config.txt_ telling the Pi to active it.

This is done by appending a device tree parameter to the end of the file.

```
~ # tail -1 /boot/config.txt
dtparam=i2c_arm=on
```

A restart is needed to activate the change.

## Loading modules for I2C
The kernel module _i2c-bcm2835_ is used to access the I2C bus, _i2c-dev_ is used to create _/dev/i2c-1_, and _rtc-ds1307_ will load support for the ds1307 (of which the ds3231 is a compatible, more accurate version.)

Loading from command-line looks like this:

```
/sbin/modprobe i2c-bcm2835
/sbin/modprobe i2c-dev
/sbin/modprobe rtc-ds1307
```

Using the command `i2cdetect 1` will probe all the devices on I2C bus one, and should show your clock module address in a table. (Mine was at address 0x68.)

## Instantiating the device
Once the address is known, we need to tell the kernel what kind of device it is. We do this by writing the model and address to a file in the _/sys_ pseudo file system. This is shown below for a DS1307 compatible clock module at I2C address 0x68.

```
echo "ds1307 0x68" > /sys/class/i2c-dev/i2c-1/device/new_device
```

## Setting the time
Chances are good a brand new clock module will not have the correct date and time. We can use a combination of the _date_ command and the _hwclock_ command to fix this.

First, check the RTC's time.

```
~ # hwclock
Thu Jan  1 00:06:49 1970  0.000000 seconds
```

Next, set the system time to UTC time.

```
date -s '2026-01-03 08:49:00'
```

Finally, write the system time to the RTC.

```
hwclock -w
```

Check the RTC again to make sure it's correct.

## Configuring _rcS_
So far everything looks good. The date and time are correct. But everything will be back Jan 1, 1970 if we don't make sure the clock is available at boot. Adding the following lines to _/etc/init.d/rcS_ will do the trick.

```
# Enable I2C hardware clock
/sbin/modprobe i2c-bcm2835
/sbin/modprobe i2c-dev
/sbin/modprobe rtc-ds1307
echo "ds1307 0x68" > /sys/class/i2c-dev/i2c-1/device/new_device
```

It will need to go after the _/sys_ file system is mounted, because we need to write to a file in the _/sys_ hierarchy. I inserted the lines after mounting pseudo filesystems, but before starting syslogd and klogd.

## Testing with a reboot
I was expecting to need a `hwclock -s` somewhere in _rcS_, but interestingly the time remains correct without it.

## Next steps
It would be nice to use the RTC module as a poor man's network time server. But unfortunately, I'm not able to get the BusyBox ntpd to start listening (at least it's not showing up in `netstat -tln` output. I think this is due to ntpd not being able to sync to an upstream peer, because the Pi is not internet connected. My guess is it needs to sync itself before it offers time services to others.
