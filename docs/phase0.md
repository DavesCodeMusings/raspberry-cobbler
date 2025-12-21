# Phase 0: Setting up the Ubuntu development virtual machine
I'm using an Intel-based Windows laptop as my development host, but I'm doing the actual work on an Ubuntu virtual machine (VM). This helps keep the instructions consistent. It doesn't matter if your host OS is Windows, Mac, or Linux. As long as you can run Ubuntu as a virtual machine, our experiences should be the same.

> In this guide, I will refer to the development machines like this:
> * Development host -- the laptop running the virtual machine hypervisor.
> * Development VM -- the virtual machine running Ubuntu.
>
> Different system prompts will be used to show which machine commands are to be run on.
> * `PS>` is PowerShell on the development host
> * `you@@Ubuntu:~$` is BASH on the development VM
> * `~ #` is /bin/sh on the Raspberry Pi

## Choosing a hypervisor
I'm using VirtualBox as my hypervisor. You can use whatever you're comfortable with. The only difference will be the way the Raspberry Pi microSD card is made available to the virtual machine.

If you've never used VirtualBox, you'll need to visit https://www.virtualbox.org/ to download it. A typical install is fine for what we're doing here. Since we're only running the one Ubuntu development VM, our CPU, RAM and storage impact on the development host will be minimal.

> If you're using a development host that won't support virtualization software (or you just don't want to deal with it), have a look at [README_BINARIES.md](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/README_BINARIES.md) for information on precompiled arm64 versions of the software packages used in the project. 

## Setting up the virtual machine
This is what the development VM will need:
* 1 CPU
* 2G RAM
* 25G virtual disk
* Bridged network adapter

How you create this VM will depend on the hypervisor you're using. In VirtualBox, you simply select _Machine > New_ from the menu.

> Note: Changing the default networking from NAT to bridged allows you to make an SSH connection to the virtual machine. It's not required, but it does make it easier to cut and paste commands.

## Installing Ubuntu
You'll need to get the Ubuntu ISO image from https://ubuntu.com/download/server and use it to instll the development VM operating system.

> With VirtualBox, I'm using the _Unattended Guest OS Installation_ feature to preconfigure parameters and let things run uninterrupted. Installation takes a while, so this is a good time to make lunch.

## Configuring Ubuntu for SSH
Ubuntu 24.04 LTS does not come with SSH by default. We'll need to install it to make SSH connections from the development host to the Ubuntu virtual machine. This is not strictly neccessary, but using an SSH client offers easier cut and paste capability than the VirtualBox console.

Log in via the virtual machine console and run the following:

```
sudo apt-get update
sudo apt-get install ssh
ip addr show
```

Test the SSH connection using the Ubuntu server's IP address shown by the `ip addr show` command.

## Making the microSD card available
By default, the microSD card on the development host will not be seen by the virtual machine. It has to be attached as a USB device. (Even the built-in card reader on my laptop shows up as a USB device.)

Here's how to do it in VirtualBox:
1. From the Virtualbox virtual machine, select _Devices > USB_ from the menu.
2. Note the USB devices listed.
3. Exit the menu.
4. Insert the microSD card.
5. Revisit _Devices > USB_ to determine what's changed (for me it was the appearance of _Generic USB2.0-CRW_ in the menu.)
6. Select the new device to connect it to the virtual machine.

You can verify success by logging into the Ubuntu virtual machine and running `sudo dmesg`. Look at the last few lines to find the device name for the microSD card. It will most likely be _sdb_ (SCSI Disk B).

## Next steps
With the Ubuntu development VM ready, we can get to building the Pi OS.

To quote Dave Grohl of the Foo Fighters on the song All My Life...

_Done! Done! And I'm on to [the next one!](phase1.md)_
