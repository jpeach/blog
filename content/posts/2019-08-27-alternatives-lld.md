---
title: "Using alternatives(8) to enable lld"
date: 2019-08-27T19:20:01+10:00
tags: [llvm,linux,fedora]
featured_image: ""
description: ""
---

In this post, I remember how to use the
[alternatives(8)](https://linux.die.net/man/8/alternatives) mechanism
to make clang's [lld](https://lld.llvm.org) linker the default.

First, tell alternatives that `lld` is available and set it at a high priority:
```
$ sudo alternatives --install /usr/bin/ld ld /usr/bin/lld 80
$ sudo alternatives --auto ld
```

Then, just verify that it worked:
```
$ alternatives --display ld
ld - status is auto.
 link currently points to /usr/bin/lld
/usr/bin/ld.bfd - priority 50
/usr/bin/ld.gold - priority 30
/usr/bin/lld - priority 80
Current `best' version is /usr/bin/lld.
$ ld --version
LLD 6.0.1 (compatible with GNU linkers)
```
