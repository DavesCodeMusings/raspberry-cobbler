# Phase 2: Set up the Ubuntu machine for building arm binaries
This will be a short section, but neccessary to allow a fast, non-arm host machine to compile binaries for the Raspberry Pi.

## Set up Ubuntu for 64-bit arm cross compiling
The following commands will get the Ubuntu virtual machine ready to compile Busybox for 64-bit arm.

```
$ sudo apt-get update
$ sudo apt-get install gcc make gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
```

## Test compiling with the classic hello world example
1. Use your favorite editor to create the _helloworld.c_ file shown below.
2. Compile for x86-64 (Intel) with: `gcc helloworld.c -o helloworld-x86_64`
3. Compile for aarch64 (arm) using: `aarch64-linux-gnu-gcc helloworld.c -o helloworld-aarch64 -static`
4. Verfiy there are no compiler errors.

```
#include<stdio.h>
int main()
{
    printf("Hello World!\n");
    return 0;
}
```

You can check the intended architecture for each of the binaries using the Linux _file_ utility. The example below shows the differences.

```
dave@Ubuntu:~$ file helloworld-x86_64
helloworld-x86_64: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c6133d7ba92a5306d0b2ad6ac35703ae6b7113d3, for GNU/Linux 3.2.0, not stripped
dave@Ubuntu:~$ file helloworld-aarch64
helloworld-aarch64: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=bf8e456da7d58cd4dbd9003d306cf886e09acf41, for GNU/Linux 3.7.0, not stripped
```

Only the x86-64 version will run on the development VM. The aarch64 hello world results in an error.

```
dave@Ubuntu:~$ ./helloworld-x86_64
Hello World!
dave@Ubuntu:~$ ./helloworld-aarch64
-bash: ./helloworld-aarch64: cannot execute binary file: Exec format error
```

Reference: https://jensd.be/1126/linux/cross-compiling-for-arm-or-aarch64-on-debian-or-ubuntu

# Phase 2 Review
At this point, our Ubuntu development machine should be able to compile binaries for both its native CPU architecture and for the Raspberry Pi 3's 64-bit arm. We were able to test this by examining the two files and trying to run them.

Seeing _Hello World!_ displayed is not very exciting, but it does prove that we can use a fast machine to compile instead of trying to set up development tools on the resource constrained Raspberry Pi.

# Next Steps
Now that we have the foundation laid, with the boot loader, the kernel, and the Ubuntu VM set up to compile arm64 binaries, we can get on to the root file system. And this, my friends, is where the magic happens.

Because if De La Soul and Schoolhouse Rock taught us anything, it's that [phase 3 is a magic number](phase3.md)
