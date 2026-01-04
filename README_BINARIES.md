# Precompiled arm64 binaries
In the repository, you will find files with names like _program_.arm64.tar.gz. These are compiled versions of the programs used in the guide. So, if you're not able to get cross-compiling working with the Ubuntu virtual machine, you can still follow along.

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

## Side quest binaries
Some of the side quest documents have binary packages that go with them as well. You can either build them yourself using the Ubuntu VM'scross compiler, or you can look for a pre-compiled version.

Precompiled packages are built using GitHub Actions and stored as workflow artifacts. As a result, you'll have to go to the latest run of the job to find the file.

Look under Actions for workflow runs named like "_package_ arm64 build". Click the workflow run to find the file artifact.

Side quest binaries are in a .tar.zip (as opposed to gzip) because GitHub only does artifacts as .zip files. BusyBox comes with an _unzip_ command, so this should not be too much hardship.
