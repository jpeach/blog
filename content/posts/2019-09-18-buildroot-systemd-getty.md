---
title: "Buildroot, Systemd, and Getty"
date: 2019-09-18T16:38:13+10:00
tags: [buildroot,systemd,getty,linux]
featured_image: ""
description: ""
---

I just spent a few hours trying to get systemd to spawn a getty on the vt
console for a Linux image that I was building with Buildroot. It turns
out that it helps to read the right documentation, which in this case
was a blog about systemd
[console handling](http://0pointer.de/blog/projects/serial-console.html).

The money quote for me was:

> In systemd, two template units are responsible for bringing up a
> login prompt on text consoles:
> 
> getty@.service is responsible for virtual terminal (VT) login
> prompts, i.e. those on your VGA screen as exposed in /dev/tty1
> and similar devices.
> 
> serial-getty@.service is responsible for all other terminals,
> including serial ports such as /dev/ttyS0. It differs in a couple
> of ways from getty@.service: among other things the $TERM environment
> variable is set to vt102 (hopefully a good default for most serial
> terminals) rather than linux (which is the right choice for VTs
> only), and a special logic that clears the VT scrollback buffer
> (and only work on VTs) is skipped.


My Buildroot config wired up a serial-getty but not a getty. Once I
added the `getty@tty1.service`, systemd did what I wanted. Next step is
to understand how to control this in Buildroot ...
