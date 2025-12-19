# Precompiled arm64 binaries
You will find files here with names like _program_.arm64.tar.gz. These are compiled versions of the programs used in the guide. So, if you're not able to get cross-compiling working with the Ubuntu virtual machine, you can still follow along.

You will need some sort of Linux machine to extract the files, because Windows cannot handle the symbolic links used.

Once you mount your microSD root file system on /mnt, you can do this:

```
cd ~
git clone https://github.com/DavesCodeMusings/raspberry-cobbler.git
cd /mnt
tar -zxf ~/raspberry-cobbler/busybox.arm.tar.gz
```

Replace busybox.arm.tar.gz with the appropriate file name for the project phase you're working on.

Licensing information can be found in the /usr/share/doc directory of each package.
