---
title: "Remember to set CONFIG_CFS_BANDWIDTH"
date: 2019-10-29T00:00:00+10:00
tags: [linux,containers,runc]
featured_image: ""
description: ""
---

I spent a while trying to debug a runc problem where it would always get
an `EACCES` error writing the `cpu.cfs_period_us` file in a cpu cgroup.

The problem turned out to be that I had not enabled `CONFIG_CFS_BANDWIDTH`
in my kernel build. Presumably, when runc tries to write the file,
it passes `O_CREAT` and cgroupfs doesn’t let it create a new file,
which leads to the somewhat surprising error.

So, if you get this error, just turn on `CONFIG_CFS_BANDWIDTH` :)
