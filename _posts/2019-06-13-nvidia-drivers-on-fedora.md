---
permalink: /:categories/:title/
layout: post
title: "Debugging: Installing NVIDIA Drivers on Fedora"
date: 2019-06-13 11:00:00.000000000 +00:00
type: post
published: true
status: publish
categories:
- mini-projects
image: https://owenjonesuob.github.io/assets/featured/fedora_nvidia.jpg
tags:
- linux
- fedora
- nvidia
- debugging
excerpt: "TL;DR: I had mega issues with getting NVIDIA graphics to work on Fedora. This is a play-by-play of my debugging process, and ultimately the solution turned out to be rather simple."
---

**TL;DR:** I had mega issues with getting NVIDIA graphics to work on Fedora. This is a play-by-play of my debugging process, and ultimately the solution turned out to be rather simple.

---

## Context

Sometime last year my company decided to upgrade its IT assets, and consequently sold off a bunch of old laptops to interested employees.

I had been toying with the idea of getting a spare laptop anyway in order to play around with a Linux-based operating system, so this was the perfect opportunity. I managed to nab myself a chunky old Dell Latitude and, much to the frustration of my friend in IT who had spent the afternoon setting up a fresh Windows installation for me, asked him to borrow a USB stick and then immediately wiped the hard drive so that I could install Fedora.

Immediately, however, karma struck. I was attempting to use Fedora Media Writer to install the OS onto the laptop; but every time I successfully booted to the installation GUI, the laptop would freeze completely and I'd have to turn it off via the power button.

Eventually I succeeded by means of the installer's "basic graphics mode". However, after rebooting, it became clear that this had come with a catch: my screen was now stuck at a low resolution. Of course, I wanted to take advantage of all 1080 of my available p's, so set about looking for the issue.

Bear in mind that this was more or less my first experience with a Linux OS. I knew how to do simple command line things, but I didn't really know how Linux was set up, or how to change boot parameters, or what on earth a GRUB was, or any of those other fun low-level system things. So after a LOT of searching and reading, I eventually discovered the `/etc/default/grub` file and the `nomodeset` boot parameter. "Hah, problem solved!" I exclaimed, exterminating `nomodeset` from wherever it appeared. I rebooted the laptop: it froze immediately.

By this time I'd had enough for one day, so I shut the lid and forgot about it.


---

## Diagnosis and solution

Fast-forward an ENTIRE YEAR and finally, I had the time and inclination to take another look at the problem. For good measure, I decided to start with a fresh installation of the latest version the OS, Fedora 30.

Once again I had to run the installer in "basic graphics" mode, and so I was faced with lower-than-ideal screen resolution.

I started by upgrading the OS with a `sudo dnf update`, which took a while, as expected.

Then I spent several days trying ALL SORTS of fixes and tweaks and changes that I found on the internet. The laptop contains a NVIDIA graphics card, and apparently there are a _whooooole_ load of issues with using Fedora and GNOME display manager (Fedora's default graphical system) on NVIDIA hardware. I added repositories and installed drivers. I changed boot parameters. I modified configuration files. I added things to blacklists, and removed things from blacklists, and installed different display managers, and regenerated initramfs, and banged on the desk while laughing hysterically, and all sorts of other weird and wonderful things; but all to no avail.

On the plus side, I was learning a _lot_ about Linux. I learned about virtual terminals when I accidentally disabled GDM and ended up with a bootscreen which perpetually flashed "Starting GNOME display manager". I discovered the GRUB boot menu, and runlevels when I got fed up with trying to type my password letter by letter whenever the bootscreen flashed up for half a second at a time.

Eventually I ended up wiping and reinstalling the OS one more time, and then sat down with a cup of tea in front of a runlevel-3 command line, determined to get to the bottom of the issue once and for all, with logic and patience.

First things first: make sure the NVIDIA graphics are actually being used, right? So I checked the list of loaded modules for anything containing "nvidia":

```
lsmod | grep nvidia
```

Oh. Nothing came back from that.

Ah yes, I thought. Duh. I'll need to install the NVIDIA drivers.

There seem to be a couple of choices for where to get those drivers, but the repository which seemed to come up most often was RPM Fusion. So first we add the repo, then we install the driver according to the instructions on their website (https://rpmfusion.org/Howto/NVIDIA):

```
sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf install xorg-x11-drv-nvidia akmod-nvidia
```

Cool. Reboot and try again.

```
lsmod | grep nvidia
```

Nothing, nada, zilch. I double-checked that the NVIDIA driver had installed:

```
modinfo nvidia
```

That gave me a bunch of info, so the NVIDIA driver was definitely available... so if not NVIDIA, what _was_ being used for graphics?

I remembered reading something about the fallback graphics system, which would be used if the desired system failed to load properly. Suspecting that this might be the case, I made sure:

```
lsmod | grep nouveau
```

This time I got several lines of output! So the problem wasn't that the NVIDIA graphics module was misbehaving, but rather that it wasn't loading at all.

This was progress! Cue a lot more playing around with boot parameters etc...

Eventually I realised that I could just force the system to load the NVIDIA module right there and then, using the `modprobe` command:

```
modprobe nvidia
## modprobe: ERROR: could not insert 'nvidia': no such device
```

Wait. No no no wait, what?! But `modinfo` said that oh you I just aaaaaaaaaaaaaaaaaaAAAAAAAAAAAAAAAAAAAA (┛✧Д✧)┛彡┻━┻

One quick cuppa later, gathered and composed, I tried again and asked for a little bit more information:

```
modprobe nvidia -v
```

I had also recently come across `dmesg` for displaying kernel messages, so thought I'd try that out too...

```
dmesg | less
```

Ah. Some wild NVIDIAs appeared pretty near the bottom of that output... let's have a closer look...

```
## NVRM: The NVIDIA NVS4200M GPU installed in this system is
## NVRM:  supported through the NVIDIA 390.xx Legacy drivers. Please
## NVRM:  visit http://www.nvidia.com/object/unix.html for more
## NVRM:  information.  The 430.14 NVIDIA driver will ignore
## NVRM:  this GPU.  Continuing probe...
## NVRM: No NVIDIA graphics adapter found!
```

Oh oh oh oh oh oh oh oh oh oh oh oh oh OH.

So if I just...

```
sudo dnf remove xorg-x11-drv-nvidia
```

... and then...

```
dnf list --repo rpmfusion-nonfree | grep nvidia | grep 390
```

... and then...

```
sudo dnf install xorg-x11-drv-nvidia-390xx
```

One reboot later, and EVERYTHING WAS FINE. Everything. No freezing. No digging around in boot config files. And it's been fine ever since. It's like nothing ever happened.


---

## Review

Really this was all just rather frustrating at the time. I will freely admit that on more than one occasion during this saga I loudly exclaimed "I hate computers" to the little stuffed toy cat that sits on my desk. I say that quite often, as friends and colleagues will testify, and I stand by it. I really do hate them with a burning passion.

However, if there's one thing I hate more than computers, it's giving up on something once I've gotten my teeth into it. Even in the depths of despair, I could recognise that I was at least learning a lot of potentially useful information; and ultimately, that was enough to keep me going until I eventually hit that `dmesg` breakthrough.

I suppose it was a valuable experience, maybe, perhaps, in some ways. Maybe. Anyway, I figured it was worth writing about in case anyone is having similar issues, or in case anyone wanted to read a heartwarming story about courage in the face of adversity or something.
