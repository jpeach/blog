---
title: "Building Mesos on OS X"
date: 2015-04-16T15:28:08+10:00
tags: [mesos,macos,osx]
featured_image: ""
description: ""
---

So when you build Mesos on OS X, you have to use Homebrew to install a
bunch of dependencies. I was momentarily stumped by the fact that linking
the apr package with `brew link --force` seemed to not make the headers
available. Then I realized that you are supposed to use apr-1-config to
find the headers location.

Like this:

```bash
$ ./configure --prefix=/opt/mesos --with-apr=$(apr-1-config --prefix)
```
