# Configuring multi-user access
So far, the system boots to a shell prompt with root user privileges. It's a serial based console, so as long as you can maintain physical security, it's not a major concern. But now that we've added access over the network with SSH, it's time to re-examine our approach.

Taking a cue from mainstream Linux distributions like Raspberry Pi OS, we see that at least one non-root user is required when the system is set up. This is the account used for logging into the system. Any administrative tasks are done using _sudo_.

BusyBox doesn't provide _sudo_, but it does have _su_, which is close enough for our minimalist OS needs.

## Setting up a place for home directories
Users need a place to store their files. Most Linux distros use /home for this. We'll use it too.

```
mkdir /home
```

## Setting up a new user
BusyBox gives us the `adduser` command for the task of setting up a new account. With no command-line options, it will do the following tasks for us:

* Add the user account to /etc/passwd
* Add a group to /etc/group
* Configure the account to use /bin/sh as a shell
* Fill in _Linux User_ as the account's name
* Prompt for an initial password

Most of the defaults are fine, but we can use the `-g` option to specify something other than "Linux User" for the name.

### Adding an admin user
The example below shows adding an _admin_ user to the system.

```
~ # adduser -g 'Administrator' admin
Changing password for admin
New password:
Retype password:
passwd: password for admin changed by root
```

### Verifying what was changed
This part is purely educational and you won't need to do it every time you set up a new account. 

~ # grep admin /etc/passwd
admin:********:1000:1000:Administrator:/home/admin:/bin/sh
~ # grep admin /etc/group
admin:x:1000:
~ # ls -ld /home/admin
drwxr-sr-x    2 admin    admin         4096 Jan  1 11:31 /home/admin

We can see the new user has an account in /etc/passwd, a group in /etc/group, and a home directory of /home/admin. These tasks will be done for us anytime we set up a new user account with `adduser`.

## Testing SSH for the admin user
The reason for creating another user was to have something besides _root_ for SSH logins, so let's make sure this new user can log in with SSH and start a superuser shell.

> Spoiler: this will fail.
 
```
PS> ssh 192.168.2.100
admin@192.168.2.100's password:
~ $ su -
Password:
PTY allocation request failed on channel 0
shell request failed on channel 0
```

We're back to our old _PTY allocation request failed_ error. Didn't we fix this already by mounting _/dev/pts_?

Yes, but mdev broke it.

### What's wrong with mdev?
When we added the line `/sbin/mdev -s` to _rcS_, we handed over management of /dev to mdev. This includes the permissions on the contents of /dev. The problem we're encountering is our non-root user account does not have permission to /dev/ptmx, but it should. mdev has set the permissions to a default state.

When root was the only user on the system, the problem never surfaced, because root has access to everything.

### How do we fix mdev?
The _/etc/mdev.conf_ file we created to manage hotplug events for the network interface can also manage permissions for devices in /dev. Adding a line for _ptmx_ will let us put the permissions back to read-write for everyone. While we're at it, there are a number of other device nodes that need fixing as well.

Here's the new _/etc/mdev.conf_ with the permission fixes prepended. _ptmx_ is on line 7. The rest are standard devices everyone should be able to access, except for the console which is limited to the root user. (The hotplug action for eth0 is unchanged.)

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
```

Run the `mdev -s` command to update with the new configuration or restart the Pi.

## Re-testing SSH for the admin user
We should be able to log in via SSH now as a non-root user.

## Disallowing root over SSH
Now that we can log in as a non-root user, we can disable SSH for root. This is a fairly common security enhancement and allowing root logins over SSH is rare.

Dropbear's `-w` option lets you prevent root logins via SSH. This will have the effect of limiting superuser (root) access to the console or logging in as a standard user (like admin) and using `su -` to get a root shell.

Since dropbear is controlled by _inetd_, we'll add the `-w` option there and tell _inetd_ to reread its configuration.

```
~ # cat /etc/inetd.conf
22      stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i -w
~ # killall -HUP inetd
```

## Creating additional users
You can create user accounts for other people the same way you created the _admin_ account. You'll need to be _root_ to do it.

For example, starting as _admin_:

```
~ $ su -
Password:
~ # adduser -g 'Proper Dave' dave
```

## Creating service accounts
There are also a number of user accounts reserved for system use. At a minimum, we'll create one for the "nobody" user with a UID of 65534. But, trying to run `adduser` for this UID results in an error _adduser: number 65534 is not in 0..60000_, so we'll have to append to the files directly. 

```
~ # echo "nobody:x:65534:65534:Nobody::/usr/sbin/nologin" >> /etc/passwd
~ # echo "nobody:x:65534:" >> /etc/group
```

The nobody account is an exception. Other service accounts can be created with _adduser_ almost like any other account. We'll just need to supply command-line options to override the default home directory and login shell.

Run the command below to set up a _www-data_ user. Use built-in _adduser_ help to get an understanding of all the options used.

```
~ # adduser -g 'Web Data' -D -h /srv/www -H -s /usr/sbin/nologin -u 33 www-data
~ # adduser --help
```

## Verifying service account behavior
Despite our best attempts, the service account _www-data_ should not be able to log into the system. We should not be able to start a shell as the service account either.

```
PS> ssh www-data@192.168.1.100
www-data@192.168.2.100's password:
Permission denied, please try again.
PS> ssh admin@192.168.1.100
admin@192.168.2.100's password:
~ $ su - www-data
Password:
su: incorrect password
~ $ su -
Password:
~ # su - www-data
This account is not available
~ #
```

## Phase 7 review
We have created the /home directory needed for additional users and set up a non-root account we can use for SSH logins. We also created some non-interactive service accounts with the _nobody_ and _www-data_ users. Any additional users or system accounts can be created following one of these same procedures.

## Next steps
Now that we have multiple ways to access the system, it might be nice to have a way to track what's going on. For that we can use syslog logging, and that's the [next phase](phase8.md) of the project.
