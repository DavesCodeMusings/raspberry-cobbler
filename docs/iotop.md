# iotop Side Quest
Why is my Pi's activity light (the green LED) always pulsing?

That's the question that started me on this side quest. I wanted to know what was accessing my microSD on a consistent basis. After all, I put _/tmp_ and _/var/log_ on RAM-based _tmpfs_. What was writing to persistent storage?

To find out, I'm using _iotop_.

## Installing iotop
The arm64 package of _iotop_ is available in [this repository](https://github.com/DavesCodeMusings/raspberry-cobbler). If you want to build it yourself, take a look at the GitHub Action [workflow definition](https://github.com/DavesCodeMusings/raspberry-cobbler/blob/main/.github/workflows/iotop.yml) for hints on how to do it.

You can install the tar.gz using one of the same methods covered in the guide.

Personally, I'm partial to SFTP the archive to _/tmp_ on the Pi, then log in and extract.

```
PS> sftp admin@192.168.1.100
cd /tmp
put iotop.tar.gz
```

```
~ # cd /
~ # tar -zxf /tmp/iotop.tzr.gz
```

## Running itop
The program has some command-line options that are handy. I settled on `-a` for _accumulated_ and `-o` for _only_. This means I will only see processes actively writing to the microSD and it will be shown in total bytes written, not bytes per second.

I've been running _iotop_ as I write this and I see 140K of writes over about 42 minutes. All of it is from _jbd2/mmcblk0p2-8_. Looking this up on the net, I found out _jdb2_ is the journalling process for ext4 filesystem. Since journalling is what reduces the chance for data corruption, I can accept it writing to the microSD.

```
T R:   0.00B         W: 140.00K ||||||||C R:   0.00B         W:   0.00B  |
TID     PRIO USER      DISK READ   DISK WRITE  GRAPH[R+>COMMAND    [T](00:41:47)
   98   be/3 root         0.00 B    140.00 K   |         [jbd2/mmcblk0p2-8]    
```

## Conclusions
After a while _jbd_ seems to calm down and actually disppears from the _iotop_ list, leaving it empty. Making an SSH connection puts _dropbear_ up on the list for a bit.

Interestingly, I tried to stress it a little with an SFTP upload of a 5 MB package file. But, instictively, I uploaded to _/tmp_ and saw nothing (because _/tmp_ is a _tmpfs_ RAM-based file system.)

So it seems the design of the system is working out, with the directories like _/tmp_ and _/var/log_ (those that recieve the most storage writes) on _tmpfs_ to save wear on the microSD.
