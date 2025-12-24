# WiFi Side Quest

**Warning: This is incomplete and perhaps incorrect!**

After finishing all the project phases you may be wondering why the only networking is wired Enternet. The Raspberry Pi 3 has wireless built in, why not use it?

The answer is, it's not trivial to set up. But if you're really motivated, keep reading.

## What makes wifi so difficult?

* Joining a wifi network
* Dealing with dynamic addresses (DHCP)
* Having the correct kernel modules and firmware

## Adding modules and firmware
Wired Ethernet was easy to set up, because the hardware drivers were already included as part of the Linux kernel (kernel8.img) we got from the Raspberry Pi firmware repository. The _eth0_ interface just appeared and we were able to give it an IP address.

The _wlan0_ wireless interface is nowhere to be seen. We need to get the interface driver working first, and that takes a few steps.

* Copy kernel modules to the Pi
* Copy the wireless adapter's firmware to the Pi
* Configure so the _wlan0_ adapter appears at boot.

### Adding modules
Fortunately, the kernel modules we need are included in the Raspberry Pi Firmware git repository we already fetched when we needed the Linux kernel.

After cloning the repo to our development VM, we found the kernel image in the _~/firmware/boot_ directory. This time we need to look in _~/firmware/modules_

There are a number of subdirectories under modules and we only need one. The key to finding out which one is to run `uname -r` on the Pi and select the module subdirectory with the same name.

```
~ # uname -r
6.12.62-v8+
```

The example above means we need the _~/firmware/modules/6.12.62-v8+_ directory.

Use one of the file transfer methods we used before to copy this entire contents of the directory as a subdirectory of _/lib/modules_ on the Pi. When you're done, it should look like the example below.

```
~ # ls -F /lib/modules/
6.12.62-v8+/
```

### Adding firmware
Copying the firmware files to the Pi is similar in many ways to copying the kernel modules. The exception is, the firmware comes from another git repository.

First, get the firmware to the development VM.

```
you@Ubuntu:~$ git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
```

Then, copy what's needed to the Pi. (But things are tricky, because of symlinks.)

Assuming the Pi's root file system is mounted on the development VM's _/mnt_ directory, and the kernel firmware repository was cloned to the development VM's _~/linux-firmware_ directory, it would be like this:

```
you@Ubuntu:~$ sudo mkdir /mnt/lib/firmware/cypress
you@Ubuntu:~$ sudo cp ~/linux-firmware/cypress/cyfmac43430-sdio.* /mnt/lib/firmware/cypress
you@Ubuntu:~$ sudo mkdir /mnt/lib/firmware/brcm
you@Ubuntu:~$ sudo ln -s ../cypress/cyfmac43430-sdio.bin /mnt/lib/firmware/brcm/cyfmac43430-sdio.bin
you@Ubuntu:~$ sudo ln -s ../cypress/cyfmac43430-sdio.clm_blob /mnt/lib/firmware/brcm/cyfmac43430-sdio.clm_blob
you@Ubuntu:~$ sudo cp ~/linux-firmware/brcm/brcmfmac43430-sdio.raspberrypi,3-model-b.txt /mnt/lib/firmware/brcm
```

When you're done, it should look like what's shown below from the perspective of the development VM.

```
you@Ubuntu:~$ ls -lR /mnt/lib/firmware/
/mnt/lib/firmware/:
total 8
drwxr-xr-x    2 root     root          4096 Jan  1 00:24 brcm
drwxr-xr-x    2 root     root          4096 Jan  1 00:23 cypress

/mnt/lib/firmware/brcm:
total 4
lrwxrwxrwx    1 root     root            31 Jan  1 00:23 brcmfmac43430-sdio.raspberrypi,3-model-b.bin -> ../cypress/cyfmac43430-sdio.bin
lrwxrwxrwx    1 root     root            36 Jan  1 00:24 brcmfmac43430-sdio.raspberrypi,3-model-b.clm_blob -> ../cypress/cyfmac43430-sdio.clm_blob
-rw-r--r--    1 root     root           874 Jan  1 00:20 brcmfmac43430-sdio.raspberrypi,3-model-b.txt

/mnt/lib/firmware/cypress:
total 420
-rw-r--r--    1 root     root        419798 Jan  1 00:20 cyfmac43430-sdio.bin
-rw-r--r--    1 root     root          4733 Jan  1 00:20 cyfmac43430-sdio.clm_blob
```

Once everything is copied over to the microSD, start up the Raspberry Pi.


## Loading the wireless adapter's kernel module

### Configuring module dependencies
The new modules are installed, but and can be loaded one by one using `insmod`, but this is tedious. Using `modprobe` is a faster way to get all the required modules loaded in one go. But first, we need to determine dependency order.

Set up module dependencies with _depmod_ as shown below.

```
~ # depmod -a
```

This only needs to be done once, when new modules are installed.

### Loading with modprobe
After configuring dependencies, a single command will now load everything needed to support the wireless adapter (kernel modules and firmware.)

```
~ # modprobe brcmfmac
```

> _brcmfmac_ is and abbreviated name for BRoadCoM Full Media Access Control, the name of the wifi chip. 

If you see _cfg80211: failed to load regulatory.db_ in the console messages, you can ignore it for now. The wifi adapter will still function, it will just limit its channel selection and output power.

### Verifying the wlan adapter appears
After loading _brcmfmac_ and everything that comes with it using the _modprobe_ command, there should be a _wlan0_ interface available to configure. 

```
~ # ls /sys/class/net/
eth0   lo     wlan0
```

## Associating to an access point
Getting the kernel modules and firmware working was only one out of three required tasks. Next, we need to get the wireless adapter associated with a wireless access point. To do this, we'll use the _wpa_supplicant_ package.

### Installing wpa_supplicant
A precompiled arm64 version of this package is available as wpa_supplicant.arm64.tar.gz in [this repository's](https://github.com/DavesCodeMusings/raspberry-cobbler) files. You can install it using one of the methods previously covered.

> You can also build from source on the development VM if you wish. The [GitHub action for wpa_supplicant](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/.github/workflows/wpa_supplicant.yml) can help you determine the steps needed.

### Setting up wifi credentials

```
~ # wpa_passphrase my-ssid > /etc/network/wpa_supplicant.conf
# reading passphrase from stdin
********
```

### Configuring the interface for dhcp

```
~ # vi /etc/network/interfaces
auto wlan0
iface wlan0 inet dhcp
```

```
~ # modprobe brcmfmac


~ # wpa_supplicant -B -i wlan0 -c /etc/network/wpa_supplicant.conf
Successfully initialized wpa_supplicant
```


```
~ # ifup wlan0
udhcpc: started, v1.38.0.git
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.1.177, server 192.168.1.1
udhcpc: lease of 192.168.1.177 obtained from 192.168.1.1, lease time 7200
```

But, wlan0 is still unconfigured!

```
~ # ifconfig wlan0

mkdir /usr/share/udhcpc

vi /usr/share/udhcpc/default.script
```

```
#! /bin/sh

# Handle DHCP events.

case $1 in
    deconfig)
        # Lease lost. Deconfigure interface, but leave it up.
        ip address flush dev $interface
        ;;
    bound)
        # Lease obtained. Configure the IP and mask.
        ip address add $ip/$mask dev $interface
        ip link set dev $interface up
        ;;
    renew)
        # Lease renewed. Refresh the parameters.
        ip addr flush dev $interface
        ip address add $ip/$mask dev $interface
        ip link set dev $interface up
        ;;
    leasefail)
        # Lease not available. Exit with warning.
        echo "Failed to obtain DHCP lease."
        ;;
esac
```


___

References
* https://raspberrypi.stackexchange.com/questions/153914/pi3s-wlan0-not-available-to-configure-in-custom-built-os
* https://packetp.com/brcmfmac-linux-wlan-driver-for-broadcom-cypress/
* https://wiki.archlinux.org/title/Wpa_supplicant

* https://git.kernel.org/pub/scm/linux/kernel/git/jberg/iw.git

> Regulatory DB
> git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/sforshee/wireless-regdb.git
