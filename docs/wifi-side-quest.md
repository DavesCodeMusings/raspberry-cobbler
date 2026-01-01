# WiFi Side Quest

**_Warning: This is incomplete and perhaps incorrect!_**

After finishing all the project phases you may be wondering why the only networking is wired Enternet. The Raspberry Pi 3 has wireless built in, why not use it?

The answer is, it's not trivial to set up. But if you're really motivated, keep reading.

## What makes wifi so difficult?

* Dealing with the added complexity of dynamic addresses (DHCP)
* Managing the authentication needed to join a wifi network
* Having the correct kernel modules and firmware for the wireless adapter

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

> The modules directory takes up about 30 MB or so of space. Not a lot, but it's nearly half the root file system on this minimalist project. If you really want to conserve space, the actual modules needed for my Raspberry Pi 3B v1.2 wifi are shown below. Other models and revisions may require different modules.
>
> ```
> ~ # lsmod | awk '{ print $1 }'
> brcmfmac_wcc
> brcmfmac
> cfg80211
> rfkill
> brcmutil
> ```


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

### Adding licensing files
The entire reason firmware exists outside of the Linux kernel is because it has different licensing than the kernel itself. So, to not risk the ire of armies of corporate lawyers, be sure to copy the firmware licenses for both Broadcom and Cypress into the microSD's firmware directory.

```
you@Ubuntu:~$ cd ~/linux-firmware
you@Ubuntu:~/linux-firmware$ cp LICENCE.broadcom_bcm43xx LICENCE.cypress /mnt/lib/firmware
```

## Booting the Pi
Once everything is copied over to the microSD, unmount the card, transfer it to the Raspberry Pi, and power up.

## Loading the wireless adapter's kernel modules
The new modules are installed, and can be loaded one by one using `insmod`, but this is tedious. Using `modprobe` is a faster way to get all the required modules loaded in one go. But first, we need to determine dependency order.

### Configuring module dependencies
Set up module dependencies with _depmod_ as shown below.

```
~ # depmod -a
```

> This command only needs to be run once, when new modules are installed.

### Loading with modprobe
After configuring dependencies, a single command will now load everything needed to support the wireless adapter (kernel modules and firmware.)

```
~ # modprobe brcmfmac
```

> _brcmfmac_ is an abbreviated name for BRoadCoM Full Media Access Control, the name of the wifi chip. 

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
A precompiled arm64 version of this package is available as wpa_supplicant.arm64.tar.gz in [this repository's](https://github.com/DavesCodeMusings/raspberry-cobbler) files. You can transfer it to the Pi using one of the methods previously covered, then extract it to the pi with `cd / && tar -zxf /path/to/wpa_supplicant.arm64.tar.gz`.

> You can also build from source on the development VM if you wish. The [GitHub action for wpa_supplicant](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/.github/workflows/wpa_supplicant.yml) can help you determine the steps needed.

### Setting up wifi credentials
Assuming you have a typical home network setup with an SSID and pre-shared key, you can feed that information to `wpa_passphrase` and it will generate a configuration file for you.
```
~ # wpa_passphrase my-ssid > /etc/network/wpa_supplicant.conf
# reading passphrase from stdin
********
```

In this case, the file is stored under _/etc/network_. The location is only important in that you'll need to tell _wpa_supplicant_ where it is using the `-c /path/to/wpa_supplicat.conf` later on.

### Manually starting wpa_supplicant to test
The command shown below will start _wpa_supplicant_ and authenticate our Raspberry Pi to the access point. At this point, we're just looking for a clean run with no errors before moving on. In other words, do not be tempted to add wpa_spplicant to _rcS_. We're not done.

```
wpa_supplicant -B -i wlan0 -c /etc/network/wpa_supplicant.conf
Successfully initialized wpa_supplicant
```

### Configuring the interface for dhcp
Technically you can use static IP addresses with wifi (in ad-hoc mode), but most people use an access point and DHCP. This makes the configuration of _/etc/network/interfaces_ very simple. Only two or three more lines are required. (The hostname line is optional.)

```
~ # tail -3 /etc/network/interfaces
auto wlan0
iface wlan0 inet dhcp
    hostname diy-pi
```

### Bringing up the interface
Let's try bringing up _wlan0_ to do test DHCP configuration.

> Spoiler: this is not going to work. The _udhcpc_ client needs a helper script to do the actual configuration.

```
~ # ifup wlan0
udhcpc: started, v1.38.0.git
udhcpc: broadcasting discover
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.1.177, server 192.168.1.1
udhcpc: lease of 192.168.1.177 obtained from 192.168.1.1, lease time 7200
```

We got an IP address. But, running `ifconfig` shows no IP address for _wlan0_

This is due to the missing helper script that _udpcpc_ expects to have available.

The script is shown below. We'll need to create the directory and the script to make DHCP work. And don't forget to set the permissions to make it executable.

```
~ # cat -n /usr/share/udhcpc/default.script
     1  #! /bin/sh
     2
     3  # Handle DHCP events.
     4
     5  # Uncomment for debugging.
     6  #echo "ip: $ip mask: $mask broadcast: $broadcast gateway: $router"
     7
     8  case $1 in
     9      deconfig)
    10          # Lease lost. Deconfigure interface, but leave it up.
    11          ip address flush dev $interface
    12          ;;
    13      bound)
    14          # Lease obtained. Configure the IP and mask.
    15          ip address add $ip/$mask dev $interface
    16          ip link set dev $interface up
    17          if ! [ -z "$router" ]; then
    18              ip route add default via $router dev $interface
    19          fi
    20          ;;
    21      renew)
    22          # Lease renewed. Refresh the parameters.
    23          ip addr flush dev $interface
    24          ip address add $ip/$mask dev $interface
    25          ip link set dev $interface up
    26          ;;
    27      leasefail)
    28          # Lease not available. Exit with warning.
    29          echo "Failed to obtain DHCP lease."
    30          ;;
    31  esac
```

What's happening here is _udhcpc_ is calling this script with a single command-line parameter to indicate the DHCP state and therefore the action required by the script.

> You may already be familiar with "release" as "renew" as terms used with DHCP. These correspond to "deconfig" and "renew" here.

## Linking it together with an mdev action

### Running wpa_supplicant automatically

```
~ # cat /etc/network/if-pre-up.d/wpa_supplicant.sh
#!/bin/sh

if echo "$IFACE" | grep -q ^wlan; then
  # /usr/sbin/iw dev wlan0 set power_save off  # uncomment to help with weak signal strength
  /usr/sbin/wpa_supplicant -B -i $IFACE -c /etc/network/wpa_supplicant.conf
fi
```

## End to end testing


## Misc clean-up

> Silencing regulatory DB warning when module loads...
> 
> ```
> git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/sforshee/wireless-regdb.git
> ```
>
> Copy all of ~/wireless-regdb/regulatory.* to /lib/firmware
>
> After boot, get / set country code with `iw reg get` / `iw reg set US` (substituting your country code.)
> 
> This only needs to be done once and will persist through restarts.

___

References
* https://raspberrypi.stackexchange.com/questions/153914/pi3s-wlan0-not-available-to-configure-in-custom-built-os
* https://packetp.com/brcmfmac-linux-wlan-driver-for-broadcom-cypress/
* https://wiki.archlinux.org/title/Wpa_supplicant

* https://git.kernel.org/pub/scm/linux/kernel/git/jberg/iw.git
