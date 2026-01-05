# The tmux Side Quest
tmux is a terminal multiplexer. If you've used it on other Linux systems, you may want to have it available on your Pi. If you're not failiar with it, don't worry, it's not required. Find out more at: https://github.com/tmux/tmux/wiki

The following steps are totally optional.

## Installing the pre-built arm64 binary package
Visit the [Raspberry Cobbler GitHub repository](https://github.com/DavesCodeMusings/raspberry-cobbler) to find a pre-built, statically-linked, arm64 binary package for tmux.

1. Visit https://github.com/DavesCodeMusings/raspberry-cobbler
2. Click Actions in the site header.
3. Find the _tmux arm64 build_ workflow in the left-hand pane and click on it.
4. Click the _tmux arm64 build_ workflow run link.
5. Look to the bottom of the page for the workflow artifacts.
6. Download the _tmux_ workflow artifact.

GitHub artifacts are in ZIP format, so it will need to be unzipped and untarred after transferring the file to the Pi.

```
~ # cd /tmp
/tmp # unzip tmux.zip
/tmp # cd /
/ # tar -xf /tmp/tmux.amd64.tar
```

You can verify by looking for _tmux_ in the _/usr/bin_ directory or use `which tmux`

## Building locale data
tmux supports multiple languages, so we'll need the locale archive available on the microSD. Otherwise, tmux will fail with a message of: _tmux: need UTF-8 locale (LC_CTYPE) but have ANSI_X3.4-1968_

We need to build a locale archive. This can be done with two commands, but it requires the _glibc_ package to be installed first. The [bash side quest](bash.md) has details on how to do that.

After _glibc_ is installed, run:

```
~ # mkdir /usr/lib/locale
~ # localedef -f UTF-8 -i en_US en_US.UTF-8
```

> If you don't want to install _glibc_, just copy _/usr/lib/locale/locale-archive_ from the Ubuntu development VM. Locale data is architecture independent.

## Installing terminfo data
tmux also needs information about various terminal types stored in the terminfo database. Otherwise, it will fail with the message: _can't find terminfo database_

You can get the terminfo data as part of the ncurses package. Download it as a workflow artifact and install it using the same procedure as installing _tmux_

## Fixing issues with Windows Terminal over SSH
If you happen to be using Windows 11 Terminal and you see strange characters when you start _tmux_, adding a longer tmux escape time may help:

```
cd ~
~ $ cat .tmux.conf
set-option -s escape-time 50
```

> This solution is from [Terminal's GitHub Issue #16384](https://github.com/microsoft/terminal/issues/16384) (which is a very solid power of 2 number, so you know it's good.)

___
_God save us, everyone, as we burn inside the fires of the tmux sun &mdash; Linkin Park (sort of)_
