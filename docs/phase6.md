# Phase 6: Adding network services
When you run Raspberry Pi OS, you can configure it to allow remote logins with Secure Shell. We can do that with this project, too. We'll take a cue from other minimalist Linux distributions and use [dropbear](https://en.wikipedia.org/wiki/Dropbear_(software)) as our SSH server.

## Build and install Dropbear
Building Dropbear from source is no more complex than building BusyBox was. But, there is a bre-built binary package in this repository that will save you time.

Just extract the .tar.gz archive onto the microSD root file system. Assuming it's mounted on the Ubunto VM's _/mnt_ directory, the commands are like this:

```
you@Ubuntu:~$ cd ~
you@Ubuntu:~$ git clone --depth=1 https://github.com/DavesCodeMusings/raspberry-cobbler.git
you@Ubuntu:~$ sudo mount /dev/sdb2 /mnt
you@Ubuntu:~$ cd /mnt
you@Ubuntu:/mnt$ sudo tar -zxvf ~/raspberry-cobbler/dropbear.arm64.tar.gz
you@Ubuntu:/mnt$ cd ~
you@Ubuntu:~$ umount /mnt
```

Be sure to eject the microSD before transferring it back to the Pi.

And, if you still want to build from source, have a look at the [GitHub Action that builds Dropbear](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/.github/workflows/dropbear.yml) for hints on how to configure and install.

## Dealing with hotplug (again!)
Because our Raspberry Pi 3's Ethernet is configured automatically by _mdev_, we are faced with some unique challenges for network services. Namely, we can't just add lines to _/etc/init.d/rcS_ to start things, because as we saw with _ifup_, the _eth0_ interface isn't neccessarily going to be available. But there is a solution.

We can handle this in two parts:
1. Configure [_inetd_](https://en.wikipedia.org/wiki/Inetd) to start _dropbear_ for incoming ssh connections.
2. Configure _ifup_ to start _inetd_ when _eth0_ (or any interface other than loopback) comes up.

### Configuring _inetd_ to start _dropbear_ for SSH
BusyBox includes _inetd_ as part of the installation. All we have to do is configure it. This is done using a combination of _/etc/services_ and _/etc/inetd.conf_ The example below will tell _inetd_ to listen on port 22 and automatically start _sshd_ when a connection is made.

```
~ # cat /etc/services
ssh             22/tcp

~ # cat /etc/inetd.conf
ssh     stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i
```

### Configuring _ifup_ to start _inetd_
Remember [back in phase 5](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/docs/phase5.md#configuring-loopback) where running _ifup_ gave warnings about missing directories? We fixed it by adding directories for _/etc/network/if-up.d_ and others. These directories exist to run custom scripts when an interface enters a certain state. The _if-up.d_ scripts are run when the interface is up.

Making the connection yet? We want to start _inetd_. We want to do this when _eth0_ is up and available.

Add a script to _/etc/network/if-up.d_ that starts _inetd_ and it should execute automatically when _eth0_ (or any other non-loopback interface) comes up... which happens when _mdev_ detects the _eth0_ network interface... which happens when the kernel detects the USB-attached network adapter as a _hotplug_ device.

Here's the script:

```
~ # cat /etc/network/if-up.d/inetd.sh
#!/bin/sh
if [ "$IFACE" != "lo" ]; then
  start-stop-daemon -S -x /usr/sbin/inetd
fi
```

Be sure to make it executable or you will be sad when it doesn't run.

```
chmod +x /etc/network/if-up.d/inetd.sh
```

> Using _start-stop-daemon_ ensures _inetd_ will only be started when it's not already running. This is important if you add a wireless or a second wired Ethernet adapter.

### Rebooting to test automatic _inetd_ start-up
Issue the `reboot` command and wait for the Pi to come back up.

Check that _inetd_ is running.

```
~ # ps -ef | grep inetd
  152 root      0:00 /usr/sbin/inetd
  154 root      0:00 grep inetd
```

And make sure _inetd_ is listening for incoming SSH connections on port 22.

```
~ # netstat -tln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
```

### Configuring Dropbear host keys
Before dropbear is ready to accept SSH connections, it needs to be configured with one or more host keys to identify itself to clients. There are three types of keys dropbear supports and we'll create one of each type.

All this can be done with the following commands:

```
~ # mkdir /etc/dropbear
~ # cd /etc/dropbear
~ # dropbearkey -t rsa -f dropbear_rsa_host_key
~ # dropbearkey -t ecdsa -f dropbear_ecdsa_host_key
~ # dropbearkey -t ed25519 -f dropbear_ed25519_host_key
```

## Testing SSH connections
> Spoiler: This is going to fail. Multiple times. It's structured to be a step-by-step learning experience. But for the impatient, we're missing the _/etc/passwd_ file and _devpts_ mounted on the _/dev/pts_directory. You'll need both those for SSH to work.

Try an SSH connection from the development host connected to the same network as the Pi. (Substitue an IP address appropriate for your network and ping first to check connectivity.)

```
PS> ssh root@192.168.1.100
Connection closed by 192.168.1.100 port 22
```

Well, it's not _connection refused_, like you'd see if the IP address were wrong, but still no shell prompt. Why?

It turns out our minimalist system, with no _/etc/passwd_, is the problem here. We're asking to log in as _root_, but the system doesn't know who root is. Creating _/etc/password_ will fix it.

### Creating a minimal /etc/passwd and /etc/group
We'll do this in two steps. First, we'll create a boilerplate _/etc/passwd_ entry with no password, then we'll use `passwd` to set the actual password.

```
~ # cat /etc/passwd
root::0:0:SuperUser:/root:/bin/sh
```

```
~ # cat /etc/group
root:x:0:
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
PS> ssh root@192.168.1.100
root@192.168.1.100's password:
PTY allocation request failed on channel 0
shell request failed on channel 0
```

There's no more _connection closed_, but now we have a new error to deal with. What's a PTY and why isn't it allocating?

### Mounting the devpts pseudo file system
The PTY allocation that failed in the connection test is because the device nodes don't exist. We don't need to create them, the kernel will do that for us, but we do need to mount the kenel's pseudo file system that contains the device nodes.

It only takes two commands.

```
~ # /bin/mkdir /dev/pts
~ # /bin/mount -t devpts devpts /dev/pts
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
     9  # Start device manager
    10  echo "/sbin/mdev" > /proc/sys/kernel/hotplug
    11  /sbin/mdev -s
    12
    13  # Mount temporary filesystems
    14  /bin/mount -t tmpfs run /run
    15  /bin/mount -t tmpfs tmp /tmp
    16
    17  # Check and mount root and boot
    18  /sbin/fsck.ext4 -p /dev/mmcblk0p2
    19  /bin/mount -t ext4 -o remount,rw /dev/mmcblk0p2 /
    20  /sbin/fsck.fat -a /dev/mmcblk0p1
    21  /bin/mount -t vfat /dev/mmcblk0p1 /boot
    22
    23  # Bring up loopback interface
    24  /sbin/ifup lo
```

### Testing (hopefully for the last time)
Try SSH once more from the devlopment host and this time, you should see a shell prompt.
```
PS> ssh root@192.168.1.100
root@192.168.2.100's password:
Failed chdir '/root': No such file or directory
/ #
```

We got a warning about the missing _/root_ home directory, but at least we're in.

Fixing the missing directory is easy.

```
install -d -m700 -o0 -g0 /root
```

> The install command above is the same as running these three commands separately:
> mkdir / root ; chmod 700 /root ; chown 0:0 /root

## Configuring httpd
Getting SSH set up was kind of painful. But with the foundation laid, configuring other services to run is just a matter of adding a line to _/etc/inetd.conf_ then creating any files and directories the service needs.

Here's an example of setting up _inetd_ to launch a web server on the unencrypted port.

```
~ # cat /etc/inetd.conf
22      stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i
80      stream  tcp     nowait  root    /usr/sbin/httpd         httpd -i -h /srv/www
```

We'll need to add a temporary test file.

```
~ # mkdir -p /srv/www
~ # echo "Testing 1 2 3" > /srv/www/index.html
```

We also need to tell _inetd_ to re-read its configuration.

```
~ # killall -HUP inetd
```

Now, point a web browser to the Pi (or use wget) at http://192.168.1.100/

You should see the message: _Testing 1 2 3_

There's also _telnetd_ and _ftpd_ included with BusyBox, along with other ancient and insecure protocols. You could configure those in _inetd.conf_ if you wanted to. But, SSH is better than Telnet and SFTP better than FTP. And SFTP is easy to add to Dropbear.

## Adding SFTP
One of the nice features of SSH is its ability to transfer files securely with _scp_ and _sftp_. Dropbear has _scp_ but no _sftp_. However, Dropbear will look in the standard location, _/usr/libexec/sftp-server_ and use the _sftp-server_ program found there.

There is a binary package included with this repository called _openssh-sftp-server.arm64.tar.gz_ In the archive is the _sftp-server_ from OpenSSH, compiled for arm64. Using OpenSSH's _sftp-server_ the commonly accepted way to add sftp support to Dropbear.

This is entirely optional, so feel free to skip it if you don't need to transfer files.

> If installing a pre-built binary feels like cheating, you can use your Ubuntu development VM to compile OpenSSH yourself. Look at the [GitHub action for building OpenSSH](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/.github/workflows/openssh-sftp-server.yml) for hints on how to do it.
>

## Checking the time
Run the _date_ command and one glaring omission in network services will be staring you in the face.

```
~ # date
Thu Jan  1 00:00:14 UTC 1970
```

The system is stuck in the 1970s, because the Raspberry Pi has no realtime clock. It relies on network time servers and these aren't configured yet.

### Configuring ntpd
BusyBox provides _ntpd_ as a Network Time Protocol (NTP) server and client. If we create a file _/etc/ntp.conf_ listing the upstream time servers, our Pi will sync its time.

Here's what the file looks like for a Pi residing in North America.

```
~ # cat /etc/ntp.conf
server 0.north-america.pool.ntp.org
server 1.north-america.pool.ntp.org
server 2.north-america.pool.ntp.org
server 3.north-america.pool.ntp.org
```

### Sharing time with the local network
Once the Pi has an accurate time source, you can have _ntpd_ provide that time as a service on the local area network. All it takes is the _-l_ (minus el) option when ntpd is run.

And while we're thinking about running _ntpd_ at start-up, lets take a look at how that would be done.

### Starting _ntpd_
We can't really rely on _inetd_ to start _ntpd_ for us. This is because _ntpd_ needs to run right away (as soon as the network is up) to get an accurate time from the upstream servers. _inetd_ only runs servers as needed when client connections are made.

To solve this, we'll copy the _/etc/network/if-up.d/inetd.sh_ file and modify it to start _ntpd_ as well. This will work, because every script in the _/etc/network/if-up.d/_ directory will be run when an interface comes up.

Here's the script:

```
~ # cat /etc/network/if-up.d/ntpd.sh
#!/bin/sh
if [ "$IFACE" != "lo" ]; then
  start-stop-daemon -S -x /usr/sbin/ntpd -- -l
fi
```

> Be sure to use `chmod +x /etc/network/if-up.d/ntpd.sh` or you will be sad when it doesn't run.

Adding this script will cause _ntpd_ to start (at the same time _inetd_ starts) when the _eth0_ interface comes up. This means the Pi will have an accurate time as soon as the network connection is available.

> _start-stop-daemon_ is used again, just like we did when starting _inetd_. But this time, there is a `-l` parameter passed to the process being started (_ntpd_). We need to add `--` before any parameters for _ntpd_ so _start-stop-daemon_ will pass them on instead of trying to interpret them. This would be equivalent to typing `ntpd -l` at the command prompt.

## Phase 6 review
The Raspberry Pi OS is starting to feel less like a toy and more like a server. We can connect via SSH. We can serve files over HTTP and SFTP. We can even use NTP to answer the age old question posed by Flavor Flav on pretty much every Public Enemy track ever laid down... _Yo, Chuck! What time is it?_

## Next steps
There are a few more details to take care of. Namely, what if you want to log in as somebody besides root? We'll look at that in the [next section](phase7.md).

___

References:
* https://codelucky.com/mdev-command-linux/
* https://github.com/mkj/dropbear
* https://manpages.ubuntu.com/manpages/noble/en/man5/services.5.html
* https://unix.stackexchange.com/questions/769340/why-dont-devtmpfs-populate-dev-pts
* https://www.openssh.org/portable.html
* https://www.ntppool.org/en/use.html
