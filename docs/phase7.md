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

### Verifying SSH works for the admin user
The reason for creating another user was to have something besides _root_ for SSH logins, so let's make sure this new user can log in with SSH and start a superuser shell.

```
PS> ssh 192.168.2.100
admin@192.168.2.100's password:
~ $ su -
Password:
~ #
```

## Disallowing root over SSH
Dropbear's `-w` option lets you prevent root logins via SSH. This would have the effect of limiting superuser (root) access to the console or logging in as a standard user (like admin) and using `su -`. It's a faily common security enhancement.

Since dropbear is controlled by _inetd_, we'll add the `-w` option there and tell _inetd_ to reread its configuration.

~ # cat /etc/inetd.conf
22      stream  tcp     nowait  root    /usr/sbin/dropbear      dropbear -i -w
~ # killall -HUP inetd

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

## Testing
Despite our best attempts, the service account _www-data_ should not be able to log into the system. But, our admin user (and other regular user accounts) can.

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
