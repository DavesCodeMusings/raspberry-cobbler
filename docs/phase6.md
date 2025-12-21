# Adding Network Services
When you run Raspberry Pi OS, you can configure it to allow remote logins with Secure Shell. We can do that with this project, too. We'll take a cue from other minimalist Linux distributions and use [dropbear](https://en.wikipedia.org/wiki/Dropbear_(software)) as our SSH server.

## Dealing with hotplug
Because our Raspberry Pi 3's Ethernet is configured automatically by _mdev_, we are faced with some unique challenges for network services. Namely, we can't just add lines to _/etc/init.d/rcS to start things, because as we saw with _ifup_, the _eth0_ interface isn't neccessarily going to be available. But there is a solution.

We can handle this in two parts:
1. Configure [_inetd_](https://en.wikipedia.org/wiki/Inetd) to start _dropbear_ for incoming ssh connections.
2. Configure _ifup_ to start _inetd_ when _eth0_ comes up.

### Configuring _inetd_ to start _dropbear_ for SSH
BusyBox includes _inetd_ as part of the installation. All we have to do is configure it. This is done using _/etc/inetd.conf_ The example below will tell _inetd_ to listen on port 22 and automatically start _sshd_ when a connection is made.

```
~ # cat /etc/inetd.conf
22      stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i
```

### Configuring _ifup_ to start _inetd_
Remember [back in phase 5](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/docs/phase5.md#configuring-loopback) where running _ifup_ gave warnings about missing directories? We fixed it by adding directories for _/etc/network/if-up.d_ and others. These directories exist to run custom scripts when an interface enters a certain state. The _if-up.d_ scripts are run when the interface is up.

Making the connection yet? We want to start _inetd_. We want to do this when _eth0_ is up and available.

Add a script to _/etc/network/if-up.d_ that starts _inetd_ and it should execute automatically when _eth0_ comes up... which happens when _mdev_ detects the _eth0_ network interface... which happens when the kernel detects the USB-attached network adapter as a _hotplug_ device.

Here's the script:

```
~ # cat /etc/network/if-up.d/inetd.sh
#!/bin/sh
if [ "$IFACE" == "eth0" ]; then
  /usr/sbin/inetd
fi
```

Be sure to make it executable or you will be sad when it doesn't run.

```
chmod +x /etc/network/if-up.d/inetd.sh
```

### Rebooting to test automatic _inetd_ start-up
Issue the `reboot` command and wait for the Pi to come back up.

Check that _inetd_ is running.

```
~ # ps -ef | grep inetd
  152 root      0:00 /usr/sbin/inetd
  154 root      0:00 grep inetd
```

And make sure _inetd_ is listening on port 22.

```
~ # netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
```

### Configuring Dropbear host keys
Before dropbear is ready to accept SSH connections, it needs to be configured with host keys to communicate its identity to clients. There are three types of keys dropbear supports and we'll create one of each type.

All this can be done with the following commands:

```
mkdir /etc/dropbear
cd /etc/dropbear
dropbearkey -t rsa -f dropbear_rsa_host_key
dropbearkey -t ecdsa -f dropbear_ecdsa_host_key
dropbearkey -t ed25519 -f dropbear_ed25519_host_key
```

## Testing SSH connections
> Spoiler: this is going to fail. Multiple times. It's structured to be a step-by-step learning experience. But for the impatient, we're missing _/etc/passwd_ and _devpts_ mounted on _/dev/pts_ and you'll need those for SSH to work.

Try an SSH connection from the development host connected to the same network as the Pi. (Substitue IP addresses appropriate for your network and ping first to check connectivity.)

```
> ssh root@192.168.1.100
Connection closed by 192.168.1.100 port 22
```

Well, it's not _connection refused_, but what the heck?

It turns out our minimalist system, with no _/etc/passwd_ is the problem here. We're asking to log in as _root_, but the system doesn't know who root is.

### Creating /etc/passwd
We'll do this in two steps. First, we'll create a boilerplate _/etc/passwd_ with no password, then we'll use `passwd` to set the actual password.

```
~ # cat /etc/passwd
root::0:0:SuperUser:/root:/bin/sh
```

```
~ # passwd
Changing password for root
New password:
Retype password:
passwd: password for root changed by root
```

If you run `cat /etc/passwd` again, you'll see the second field is now filled in with a password.

> Note: The system is using a very basic encryption algorithm and there is no shadow password file with restrictive permissions. In other words, don't use a high value password here. It's trivial to crack.

### Repeating the SSH connection test
Try logging in from the development host again.

```
> ssh root@192.168.1.100
root@192.168.1.100's password:
PTY allocation request failed on channel 0
shell request failed on channel 0
```

There's no more _connection closed_, but now we have a new error to deal with. What's a PTY and why isn't it allocating?

### Mounting the devpts pseudo file system
The PTY allocation that failed in the connection test is because the device nodes don't exist. We don't need to create them, the kernel will do that for us, but we do need to mount the kenel's pseudo file system that contains the device nodes.

It only takes two commands.

```
/bin/mkdir /dev/pts
/bin/mount -t devpts devpts /dev/pts
```

Add these commands to _/etc/init.d/rcS_, right after the lines that mount _/proc_ and _/sys_, and we'll never have to think about it again.

Here's _rcS_ with the new commands on lines 6 and 7. [_Yeah, 6-7_](https://en.wikipedia.org/wiki/6-7_meme).

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
    12
    13  # Check and mount root and boot
    14  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    15  /bin/mount -t ext4 -o remount,rw,noatime /dev/mmcblk0p2 /
    16  /sbin/fsck.fat -a /dev/mmcblk0p1
    17  /bin/mount -t vfat -o noatime /dev/mmcblk0p1 /boot
    18
    19  # Start device manager
    20  echo "/sbin/mdev" > /proc/sys/kernel/hotplug
    21
    22  # Bring up loopback interface
    23  /sbin/ifup lo
```

### Testing (hopefully for the last time)
Try once more from the devlopment host and this time, you should see a shell prompt.
```
PS> ssh root@192.168.1.100
root@192.168.2.100's password:
~ #
```

## Configuring httpd
Getting SSH set up was kind of painful. But with the foundation laid, configuring other services to run is just a matter of adding a line to _/etc/inetd.conf_ then creating any files and directories the service needs.

Here's an example of setting up _inetd_ to launch a web server on the unencrypted port.

```
~ # cat /etc/inetd.conf
22      stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i
80      stream  tcp     nowait  root    /usr/sbin/httpd httpd -i -h /srv/files
```

```
mkdir -p /srv/files
echo "Testing 1 2 3" > /srv/files/test.txt
```

Now, point a web browser to the Pi (or use wget) at http://192.168.1.100/test.txt

You should see the message: _Testing 1 2 3_

There's also _telnetd_ and _ftpd_ included with BusyBox, along with other ancient and insecure protocols. You could configure those in _inetd.conf_ if you wanted to. But, SSH is better than Telnet and SFTP is easy to add to Dropbear.

## Adding SFTP
One of the nice features of SSH is the ability to transfer files securely with _scp_ and _sftp_. Dropbear has _scp_ but not _sftp_. However, Dropbear will look in the standard location, _/usr/libexec/sftp-server_ and use the sftp-server found there.

There is a binary package included with this repository called _openssh-sftp-server.arm64.tar.gz_ In the archive is the sftp-server from OpenSSH, compiled for arm64. This is the commonly accepted way to add sftp support to Dropbear.

This is entirely optional, so feel free to skip it if you don't need to transfer files.

## Phase 6 review
It's starting to feel less like a toy and more like a server. If all you want is the ability to run a web server, uploading content with SFTP, you're pretty well set.

There is one remaining problem though. The system has no idea what time it is.

## Next steps
In the next phase we will get the Network Time Protocol (NTP) set up as a client to sync the Pi's clock. Then we'll configure it as a server to serve time to network other hosts, thus answering the age old question posed by Flavor Flav of Public Enemy... _Yo Chuck! What time is it?_

___

References:
* https://codelucky.com/mdev-command-linux/
* https://github.com/mkj/dropbear
* https://www.openssh.org/portable.html
