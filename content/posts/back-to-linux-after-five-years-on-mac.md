---
author: "Farbod Ahmadian"
title: "Back to Linux After 5 Years on a Mac"
date: "2026-09-28"
description: "After five years on a Mac, I bought a Slimbook Evo 14 and went back to Linux with Fedora, Hyprland and a minimal config."
tags:
- linux
- developer-experience
---

I started using GNU/Linux as my main OS in high school. At first it was a dual boot with Windows, but after a short time I kept only Linux, because it covered everything I needed. The last distro I remember having on my old laptop was Xubuntu. I don't remember the version, unfortunately.

Then I joined a company with a Mac-only policy. It was my first Mac, and I didn't want to keep switching between two operating systems, so I made it my main machine for both development and everyday use.

Now, five years later, I really miss my old Linux. I had full control over everything, and I could configure and see whatever I wanted. I miss the community, and the freedom to choose between package managers, window managers and distributions. I miss a minimal distribution without distractions. Basically, everything! I missed working with my Linux.

## Picking the hardware

I looked at a few options, like System76, Framework and TUXEDO Computers, and ended up buying a [Slimbook Evo 14](https://slimbook.com/en/evo).  The others didn't work out for me, either because of the import taxes from the US to Europe, or because they just cost too much for what you got.
(We're in the middle of the AI hype as I write this, and RAM prices are absurd.)

I got it with a 1 TB Samsung 990 Pro SSD and 32 GB of DDR5 RAM. The CPU is an AMD Ryzen AI 9 365.

![The Slimbook Evo box](/images/slimbook-box.jpg)

![Slimbook Evo 14 unboxed, with the user manual, a card and a sticker](/images/slimbook-unboxing.jpg)

## Fedora and Hyprland

After one week, I'm really happy with my choice, and everything looks good so far.

I installed Fedora with Hyprland and configured it with [Fedora-Hyprland](https://github.com/LinuxBeginnings/Fedora-Hyprland). I found it too fancy and too AI-looking, so I changed it to a more minimal setup that suits me better, and added it to my [dotfiles](https://github.com/farbodahm/dotfiles).

By minimal I mean I want to control everything with the keyboard and key bindings. I don't want fancy animations or fancy popups. Just the window of the application I'm using. This is how it looks now:

![My current minimal Hyprland desktop](/images/linux-desktop.png)

And I'm happy now!

Oh, and the first thing I ran into after installing it was the good old Wi-Fi issue :)) That felt like being back home.

The card was there and up, Ethernet and `nmcli` worked fine, but NetworkManager kept saying there was no Wi-Fi device. With some help from Claude, I found out it wasn't a driver problem at all. Fedora splits NetworkManager into subpackages, and `NetworkManager-wifi` is the one that handles wireless devices. My minimal install had the core daemon but not the Wi-Fi plugin, so NetworkManager couldn't see the card, no matter what I did to the interface.


```bash
rpm -q NetworkManager-wifi wpa_supplicant
```

I plugged in an Ethernet cable and ran:

```bash
sudo dnf install NetworkManager-wifi wpa_supplicant
sudo systemctl restart NetworkManager
nmcli device status
nmcli device wifi list
```

## Why not Omarchy

As I write this, [Omarchy](https://omarchy.org/) is super hyped as well. I'm not going to try it, for the same reasons I dropped the first config, and they apply to other parts of my life too: it's too hyped, too AI-driven and too fancy. I'm looking for something minimal that isn't shaped by someone else's opinions.
