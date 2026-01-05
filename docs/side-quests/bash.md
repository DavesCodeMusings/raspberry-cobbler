# Installing BASH shell
_BASH_ is available as a pre-compiled arm64 binary, but it's not statically-linked. It depends on _glibc_ and _ncurses_ to function. This requires additional overhead, but it sets us up for installing other applications that have the same shared libraries dependencies.

Here are the high-level steps:

1. Install bash, glibc, and ncurses packages.
2. Create a linker configuration file.
3. Update the linker cache.

```
~ # cd /tmp
~ # unzip *.zip
~ # cd /
~ # tar -xf /tmp/bash.arm64.tar
~ # tar -xf /tmp/glibc-2.42.arm64.tar
~ # tar -xf /tmp/ncurses-6.6.arm64.tar

~ # cat /etc/ld.so.conf
/lib
/lib64
/usr/lib
/usr/lib64

~ # ldconfig
```

## Test the new shell

```
~ # bash
bash-5.3#
```

## Create _/bin/bash_ symlink
BASH is installed as _/usr/bin/bash_. Many shell scripts will reference _/bin/bash_. To account for this, a symbolic link is set up in _/bin_.

```
ln -s /usr/bin/bash /bin/bash
```

## Clean up package files
Since _/tmp_ is a RAM-based file system, any files sitting there are taking up system memory. Deleting them will free up space.

## Next steps
_glibc_ and _ncurses_ are common requirements for a number of other software packges. If you want to explore installing additional software, these library dependencies are already installed with bash.
