# Filling in the Gaps

**_This phase may change as more gaps are identified_**

In the quest for something usable in each phase of the project, some details got glossed over. In this final phase, we'll address those shortcomings to make the system more robust.

## Controlling system start up and shutdown with inittab
So far, system start-up and shut-down just seem to happen and are not well understood. We put things in _/etc/init.d/rcS_ and they run at boot time. When we shut down, file systems are magically unmounted with no effort from us.

Most Linux systems use _systemd_ to control start-up and shutdown. BusyBox is minimalist and uses _init_ to handle these things. And _init_ is configured with a file called _/etc/inittab_. Everything that's happeneing on our system is because of how BusyBox behaves when _/etc/inittab_ is missing.

In short, it runs _/etc/init.d/rcS_ at boot, then starts a shell on the console. When the system is shut down, it executes `unmount -a -r`.

If there were an actual _/etc/inittab_, it would look something like this:

```
::sysinit:/etc/init.d/rcS
::askfirst:/bin/sh
::shutdown:/bin/umount -a -r
```

> There's a little more to it, but this is very close.

Lines in _/etc/inittab_ have the following format for BusyBox:

```
tty:runlevels:action:command
```

* tty is where the input/output is directed.
* runlevels are ignored.
* action is one of: sysinit, respawn, askfirst, wait, once, restart, ctrlaltdel, or shutdown.
* command is a script or single command to be run.

Using the example above, we can see _sysinit_ (start-up or boot) running _/etc/init.d/rcS_, so that's how things in _rcS_ get run at boot. We also see _shutdown_ and `/bin/umount -a -r` ensuring file systems are unmounted at shutdown.

There's also _askfirst_ and `/bin/sh`. This line is the reason we see _Please press Enter to activate this console._ buried in our console messsages at boot.

We can create our own _/etc/inittab_ using the example shown below.

```
~ # cat /etc/inittab
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyAMA0 115200 vt100
::shutdown:/bin/umount -a -r
```

This is slightly different in that the _askfirst_ has been replaced with _respawn_, and _/bin/sh_ has been replaced by _/sbin/getty_. This will result in a more traditional looking login prompt when the system is restarted.

Try it out, and you should see something like what's shown below.

> Before restarting, check the following to avoid getting locked out:
> * Ensure _/dev/ttyAMA0_ exists. If it does not, _getty_ will fail.
> * Verify logging in via SSH and elevating to root with `su -`

```
(none) login: root
Password:
~ #
```

## Setting the host name
In the previous step, you should be able to sign in as root. But what's with the _(none)_ in the login prompt?

This happens because there's no host name. To fix it, we need to create _/etc/hostname_ and then use it together with the `hostname` command in the system start-up script _rcS_

Here's an example of _/etc/hostname_:

```
cobbler.home
```

It's only one line.  _cobbler_ is the host portion and _home_ is the domain.

The snippet of _rcS_ below shows where the `hostname` command is inserted on line 4. This sets the host name using the contents of _/etc/hostname_

```
~ # cat -n /etc/init.d/rcS
     1  #! /bin/sh
     2
     3  # Set system identity
     4  /bin/hostname -F /etc/hostname
     5
     6  # Mount pseudo filesystems
     7  /bin/mount -t proc proc /proc
     8  /bin/mount -t sysfs sysfs /sys
     9  /bin/mkdir /dev/pts
    10  /bin/mount -t devpts devpts /dev/pts
```

Restart the system and the login prompt should now contain the host name instead of (none).

```
cobbler.home login: root
Password:
~ #
```

> You can use almost any name you want. But, if you plan to give your Pi access to the internet, be sure to choose a domain name that is not already in use. _.home_ is one such domain name reserved for private use.

## Adding the Pi to _/etc/hosts_
Now that the system has a host name, try pinging it using that name.

> Spoiler: This will fail and it will take a bit of time to do so.

```
~ # ping cobbler.home
ping: bad address 'cobbler.home'
```

The ping command (and other network utilities) work with IP addresses. So their first task is to determine the IP address for our host. And we haven't set that up yet.

To make the ping work, we need to have a way to resolve the name to an IP. The easy way is using _/etc/hosts_. Below is an example of everything configured and working.

```
~ # cat /etc/hosts
192.168.1.100   cobbler.home    cobbler
~ # ping cobbler.home
PING cobbler.home (192.168.1.100): 56 data bytes
64 bytes from 192.168.1.100: seq=0 ttl=64 time=0.202 ms
^C
```

Your IP address and host name will be different, but the format is the same: _IP address_ (whitespace) _long host name_ (whitespace) _short host name_

For consistency, it's good practice to also add localhost. A more complete example of _/etc/hosts_ is shown below.

```
127.0.0.1       localhost.localdomain   localhost
192.168.3.100   cobbler.home            cobbler
```

You can add more entries for other hosts on your network, but keep in mind this is a static mapping, so DHCP assigned addresses will not be a good fit for _/etc/hosts_.

## Identifying other known gaps
There are more things that need attention. Some are listed below.

* No terminfo data for terminal capabilities
* No locale data for multiple language support
* DNS names don't resolve to IP addresses
* No clock for offline time keeping

Most of these are addressed in "side quest" documents. The system will still function without these things being solved. But, it can always be improved.

## Review
This phase wraps up the project. We have a minimalist Raspberry Pi based system, that can be accessed over a network, and only takes about 10 seconds to boot. It can be accessed by multiple different user accounts and uses a number of security best practices, like shadow passwords and strong encryption.

## Next Steps
I'd love to read about your own project you've built with the minimalist Raspberry Pi as a starting point. Use the [Discussions](https://github.com/DavesCodeMusings/raspberry-cobbler/discussions) topic for _Show & Tell_ to give us the details.

___

References:
* https://github.com/brgl/busybox/blob/master/examples/inittab
