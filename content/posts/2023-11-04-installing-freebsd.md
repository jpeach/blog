---
title: "Installing FreeBSD network drivers"
date: 2023-11-04T12:47:25+11:00
subtitle: ""
tags: [freebsd,sysadmin]
---

So I decided to build a local NAS system to store music (since I'm worried
my CDs are starting to degrade), and to do Time Machine backups (last
backup 2018!). Despite never having used FreeBSD before, I am planning
to go with FreeBSD for this since I want to use a native well-integrated
ZFS implementation.

Anyway, the first issue I've hit is that I need RealTek network drivers, but
they are not included in the installer kernel. So there are various blog posts
around that describe copying them into the ports system using a USB stick, etc.
Fortunately, I have a USB network adapter, and FreeBSD supports that in the
default kernel.

So after booting with the USB adapter attached, manually get a DHCP lease:
```
dhclient ue0
```

Now I have some basic connectivity, RealTek drivers are apparantly available in the
[ports](https://docs.freebsd.org/en/books/handbook/ports/) system:
```
pkg
pkg update -f
pkg install realtek-re-kmod
```

But as soon as I did that, the ports package mentioned there is already
a built-in driver, so I wasn't sure whether my chipset wasn't supported
by that driver, or whether there was just some sort of misconfiguration.
Turned out that I had the RTL8125 adapter, which needs the 
[ports driver](https://cgit.freebsd.org/ports/plain/net/realtek-re-kmod/pkg-descr?revision=HEAD).

From then on, it was straightforward to follow the
[network guide](https://docs.freebsd.org/en/books/handbook/network/)
in the handbook to get networking configured.
