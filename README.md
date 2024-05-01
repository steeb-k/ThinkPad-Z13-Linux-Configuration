# Linux on the Lenovo ThinkPad Z13

The short and sweet (as of April 28, 2024): Fedora seems to be the winner here. It has so far required the least amount of finagling to get things working (particularly Howdy), it hands-down has the lowest power consumption, and has more software support than either Debian (disappointingly - as this is what all of my server boxes run) or NixOS (at least for standard repos.) Arch has as much up-to-date software, but is generally harder to configure and has more update issues. 

There is pretty much always a chance I'll return to NixOS, as it was the most reliable and configurable, but new issues are more difficult to address when they come up.  

## NixOS

On NixOS, I had power issues any time I switched off of the 6.5 kernel. I think that I may have found a fix for this on Arch, but given the amount of work it would have taken to rapidly test fixes, I likely never would have gotten there.

It was actually demoralizing to realize I had to give it up - but I just could not take it any longer. Arch is quite literally the opposite - not exactly stable, but very forgiving for small mistakes and quick to test new packages. 

## Arch Linux

I'd been using Arch as my primary OS for about five years before switching over to NixOS. At that point, I was tired of the huge amount of updates that had to be done every week (I made extensive use of the AUR and build a lot of packages from source, foolishly.)  One thing I have learned from my time on NixOS is that less is more - the bloat of unnecessary packages would create unmanageable tangles of packages.The Nix package manager was a godsend for this - the ability to just try a package in a temporary environment made so I didn't need to install bizarre esoteric software I would only use once.   
  
However, the Nix package manager still works great on Arch!

## Fedora

So, on to Fedora. Arch proved annoying, again. Whereas things were  always available, configuration and dependencies ended up being hellish. So, on to Fedora. 

Configuring Howdy in Arch was fairly straightforward with some instructions form the Arch Wiki (which also broke on an update literally the next day), but it was a different story in Fedora. After adding the howdy-beta COPR repo, installation was straightforward and face validation via sudo worked great - but it simply wouldn't work at the GDM login screen.   
  
The trick here was to setup an SELinux module as follows:

```
#Turning off SELinux
sudo setenforce 0 

#open a new terminal window and run a sudo command to kick off howdy authentication - this teaches SELinux how it works, I guess

sudo ping google 

#Turn SELinux back on
sudo setenforce 1 

#This generates a security module, which we'll import next
ausearch -c python3 | audit2allow -m howdy >howdy.te 

sudo checkmodule -M -m -o howdy.mod howdy.te

sudo semodule_package -o howdy.pp -m howdy.mod

sudo semodule -i howdy.pp
```

Source:  [linuxreviews.org](https://linuxreviews.org/Howdy/SELinux) / [archive.is](https://archive.is/Kuv8L)

## Power Management Fixes (Suspend)

**As of April 2024, if you run a kernel newer than 6.0, chances are that you *NEED* this to have sleep work properly!**

So, as mentioned earlier, all versions of the Linux kernel (with the bizarre exception of 6.5) would cause my laptop to occasionally fail to wake from sleep (or go into sleep - I never got this nailed down, since Nix made rolling back very very very simple.)  What seems to have worked (as of April 29, 2024 on kernel version 6.8.7):

```
#/etc/udev/rules.d/99-avoid-i2c-wakeup.rules

KERNEL=="i2c-ELAN06A0:00", SUBSYSTEM=="i2c", ATTR{power/wakeup}="disabled"
```

**NixOS udev rule:** Things are a little different in NixOS, since you cannot just create a udev `*.rules` file. Add the following ot your `configuration.nix` file:

```
  services.udev.extraRules = ''
    KERNEL=="i2c-ELAN06A0:00", SUBSYSTEM=="i2c", ATTR{power/wakeup}="disabled"
  '';
```

Source: [Arch Wiki](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Instantaneous_wakeups_from_suspend) / [archive.is](https://archive.is/Rup9g)

>[!TIP]
>What is this doing?
>i2c-ELAN06A0:00 is the hardware address of the haptic trackpad. Apparently, it causes the laptop to occasionally wake from sleep immediately after being told to suspend. Between that and immediately jamming the laptop into my bag when I put it to sleep, I was creating an overheating issue. Theoretically, this will solve the issue.


## Power Profile Daemon Tweaks

Some smaller tweaks I added to change the power profile when the AC adapter is plugged in or unplugged:

```
#/etc/udev/rules.d/60-onbattery.rules  

# Rule for when switching to battery
SUBSYSTEM=="power_supply",ATTR{status}=="Discharging",ATTR{capacity_level}=="Normal",RUN+="/usr/share/power-profiles/power-saver.sh" 
```

```
#/etc/udev/rules.d/61-onacpower.rules  

#Rule for when switching to AC Power 
SUBSYSTEM=="power_supply", ATTR{online}=="1", RUN+="/usr/share/power-profiles/performance.sh"  
```

```
#/usr/share/power-profiles/power-saver.sh 

#!/bin/sh 
#set performance profile through D-bus  
powerprofilesctl set power-saver  
```

```
#/usr/share/power-profiles/performance.sh  

#!/bin/sh 
#set high performance profile through D-bus  
powerprofilesctl set performance 
```

```
#Don't foget to make them executable 

sudo chmod +x/usr/share/power-profiles/power-server.sh 
sudo chmod +x /usr/share/power-profiles/performance.sh
```

# Pros and Cons

| NixOS | Arch | Debian | Fedora |
|------|------|------|------|
|✓ Easily reproduced |✓ Excellent wiki (helpful with ALL other options, too) |✓ Large package base |✓ Lots of official documentation|
|✓ Nix configs make big changes easy |✓ AUR has almost everything|✓ Apt is *really* good|✓ LOTS of pre-compiled packages|
|✗ Awful documentation|✗ Many packages need to be built from source|✗ Mostly replaced by Ubuntu, as far as broad support |✗ I don't like DNF|
|✗ Least amount of community support |✗ Frequent system-breaking upgrades|✗ Just not as supported as Fedora|✓ Everything just works with very little adjustment|
|✓ Snapshots are part of the core OS |✗ This is cowboy land - AUR packages can easily break things|**-** Package manager is reliable |**-** Package manager is reliable |
|✓ Most stable|✗ Least stable |✓ Most stable (on my servers)|✓ Supposedly very stable (my first rodeo w/ Fedora)|
