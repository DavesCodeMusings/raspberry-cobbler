# Filling in the Gaps

**_This phase is incomplete_**

## Controlling system start up and shutdown with inittab

Lines in _/etc/inittab_ have the following format:

```
tty:runlevels:action:command
```

* tty is where the input/output is directed.
* runlevels are ignored.
* action is one of: sysinit, respawn, askfirst, wait, once, restart, ctrlaltdel, or shutdown.
* command is a script or single command to be run.

A sample initab is shown below.

```
::sysinit:/etc/init.d/rcS
::respawn:/sbin/getty -L ttyAMA0 115200 vt100
::shutdown:/bin/umount -a -r
```

## Review
This phase wraps up the project. We have a minimalist Raspberry Pi based system, that can be accessed over a network, and only takes about 10 seconds to boot. It can be accessed by multiple different user accounts and uses a number of security best practices, like shadow passwords and strong encryption.

## Next Steps
I've put together some "side quest" documents that you can work through if you're interested. Otherwise, I'd love to read about your own project you've built with the minimalist Raspberry Pi as a starting point. Use the Discussions topic for Show & Tell to give us the details.

___

References:
* https://github.com/brgl/busybox/blob/master/examples/inittab
