---
layout: post
title: Upgrading My Old Laptop
date:   2019-12-26 16:25:00 +1100
categories: update
comments: true
---

Merry Christmas and happy new year!

An interesting occasion got me to dust off my nine years old laptop and install an up-to-date Linux operating system. However, the specs of my machine is pretty outdated --- an i3 CPU, 2GB of RAM, no bluetooth and a slow WiFi. The hard disk had replaced with an SSD given by a friend few years ago.

## Remove BIOS Whitelist

I did some research on replacing the WiFi module on my system and found that the BIOS had to be hacked to overwrite the manufacturer's device whitelist. This is quite frustrating, to be honest. It is non-sense to limit what user can and cannot use for the replacement of parts in the system. All compatible parts should be supported by default. If a replacement did not work, a user can always try a different solution, which is rarely the case anyway so long as the interface is the same.

It took me half a day at least to find a suitable image whose whitelist has been removed (because most of the resources online had already expired after nine years). To flash that BIOS, I need a Windows environment which I did not have. Damn it! I googled a bit and landed on one portable Windows environment that is often used as a recovery tool (that boots from USB). Writing that Windows to USB also took me two hours, figuring why `dd` was not able to mark that creation as bootable. Finally, I used a Windows virtual machine to write that recovery tool to USB. Phew... Flashing the hacked BIOS was quite smooth --- just a few clicks in my case.

## WiFi Module

My old WiFi module was a mini-PCIe type, which is replaced with a HP part that supports `a/b/g/n` and `Bluetooth 4.0` for AUD15. I am pretty happy with it because it allows me to use wireless headphones.

## CPU and RAM

As my laptop is using the HM55 chipset which supports upto the Arrandale CPUs, I only have limited CPU options and can only have at most 8GB of RAM. After some quick research, I decided to upgrade to i5-560M which is about 25% faster than my old i3-380M, because the i5 is the most cost effective option out there in the current market. I opted out the quad core i7 because of its TDP which is 45W, 15W higher than the current system.

I bought 2*4GB of RAM and an i5-560M from Taobao for about 200 RMB, which is much cheaper than what's out there on eBay. Long live Taobao.

## Cleanup and Assemble

Not too much to mention except for an extensive dust has been cleaned out of my nine years old machine. The interior design of my laptop is quite good that I would give 5 stars for fixability.

## System

I used xubuntu as xfce4 is apparently very lightweight and fast. It only uses ~400M of RAM after booting to desktop, whereas Unity uses 1GB.

![My Desktop](/assets/img/xfce-desktop.png)
![My Desk](/assets/img/workspace.jpg)