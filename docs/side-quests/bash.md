# Installing BASH shell
_BASH_ is available as a pre-compiled arm64 binary, but it depends on _glibc_ and _ncurses_ to function. This requires a few things.

1. Install bash, glibc, and ncurses packages.
2. Create a linker configuration file.
3. Update the linker cache.

```
cobbler:~# cd /tmp
cobbler:~# unzip *.zip
cobbler:~# cd /
cobbler:/# tar -xf /tmp/bash.arm64.tar
cobbler:/# tar -xf /tmp/glibc-2.42.arm64.tar
cobbler:/# tar -xf /tmp/ncurses-6.6.arm64.tar

cobbler:~# cat /etc/ld.so.conf
/lib
/lib64
/usr/lib

cobbler:/# ldconfig
```
