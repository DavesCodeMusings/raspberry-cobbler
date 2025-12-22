# The tmux Side Quest
tmux is a terminal multiplexer. If you've used it on other Linux systems, you may want to have it available on your Pi. If you're not failiar with it, don't worry, it's not required. Find out more at: https://github.com/tmux/tmux/wiki

The following steps are totally optional.

## Installing the pre-built arm64 binary package
In the [Raspberry Cobbler Git Repository](https://github.com/DavesCodeMusings/raspberry-cobbler), you will find _tmux.amd64.tar.gz_. We'll use the files in this archive to install _tmux_ on the Raspberry Pi's microSD root file system.

```
you@Ubuntu:~$ git clone https://github.com/DavesCodeMusings/raspberry-cobbler.git --depth=1
you@Ubuntu:~$ cd raspberry-cobbler
you@Ubuntu:~$ ls *.tar.gz
you@Ubuntu:~$ sudo mount /dev/sdb2 /mnt
you@Ubuntu:~$ cd /mnt
you@Ubuntu:~$ tar -zxf ~/raspberry-cobbler/tmux.amd64.tar.gz
```

## Installing locale data
tmux supports multiple languages, so we'll need the locale archive available on the microSD. Otherwise, tmux will fail with a message of: _tmux: need UTF-8 locale (LC_CTYPE) but have ANSI_X3.4-1968_

```
you@Ubuntu:~$ sudo mkdir -p /mnt/usr/lib/locale
you@Ubuntu:~$ sudo cp /usr/lib/locale/locale-archive /mnt/usr/lib/locale
```

## Terminfo data
tmux also needs information about various terminal types stored in the terminfo database. Otherwise, it will fail with the message: _can't find terminfo database_

```
you@Ubuntu:~$ sudo mkdir -p /mnt/usr/share
you@Ubuntu:~$ sudo cp -R /usr/share/terminfo /mnt/usr/share
```

## Moving the microSD back to the Pi
Similar to the other phases of the project, you'll need to unmount the microSD, detatch if from the development VM, and eject it from the host OS before moving it over.

Boot the pi and enjoy tmux!

## Fixing issues with Windows Terminal over SSH
If you happen to be using Windows 11 Terminal and you see strange characters whe you start _tmux_, adding a longer tmux escape time may help:

```
cd ~
~ $ cat .tmux.conf
set-option -s escape-time 50
```

> This solution is from [Terminal's GitHub Issue #16384](https://github.com/microsoft/terminal/issues/16384) (which is a very solid power of 2 number)

___
_God save us, everyone, as we burn inside the fires of the tmux sun &mdash; Linkin Park (sort of)_
