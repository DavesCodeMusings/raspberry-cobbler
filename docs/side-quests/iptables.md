# The iptables side quest
Keep nefarious characters from accessing your DIY OS over the network using _iptables_ to manipulate firewall rules.

## A word of warning
The Raspberry Cobbler project is meant as a learning experience, not a way to create a hardened operating system. Do not do anything foolish like attaching your Pi directly to the internet.

## Dependencies
The pre-built _iptables_ package is not statically linked. It requires _glibc_ to be installed. See the [BASH side quest](side-quests/bash.md) for help with that step.

## Installing
The pre-built arm64 package is available as a GitHub workflow artifact. See the [tmux side quest](side-quests/tmux.md) for detailed instructions on fetching biaries built as workflow artifacts.

After you have the binary, install it like this:

* Transfer the archive to the Pi's /tmp directory using SFTP
* Change directoy to /tmp and unzip the archive.
* Change directory to / and extract the .tar file
* Clean up leftover archive files

```
~ # cd /tmp
~ # unzip iptables.zip
~ # cd /
~ # tar -xf /tmp/iptables.arm64.tar
~ # rm /tmp/iptables*
```

## Testing
Use _iptables_ to list the current rules. This won't do anything other than ensure _iptables_ and the library dependencies are installed correctly. If you see something like what's shown below, it's working.

```
root@cobbler:/# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

## Creating rules
Use your favorite internet search tool to find a tutorial on _iptables_ and use it to decide what kind of firewall rules you want to create.

[GeeksForGeeks has one that covers the basics](https://www.geeksforgeeks.org/linux-unix/iptables-command-in-linux-with-examples/). Just remember to log in as root and skip all the _sudo_ nonsense.

## Saving and restoring rules
Keep in mind _iptables-save_ and _iptables-restore_ will use standard out and standard in by default. You have to use `-f` for file based save and restore.

To keep the same ruleset after reboot, be sure to save to a file after making any rule changes. Then, add _iptables-restore_ to _rcS_ so the rules can be reapplied. This can be done before network interfaces come up, unless you are targeting a specific interface with one of your rules.

___

References:
* https://www.geeksforgeeks.org/linux-unix/iptables-command-in-linux-with-examples/
* https://unix.stackexchange.com/questions/134682/iptables-add-rule-for-interface-before-it-comes-up#134685
