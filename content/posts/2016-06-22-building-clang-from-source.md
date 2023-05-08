---
title: "Miniature guide to building Clang from source"
date: 2016-06-22T15:14:30+10:00
tags: []
featured_image: ""
description: ""
---

First checkout the sources:

```bash
$ cd ~/src
$ git clone http://llvm.org/git/llvm.git
$ cd ~/src/llvm/projects
$ git clone http://llvm.org/git/compiler-rt.git
$ git clone http://llvm.org/git/libcxx.git
$ git clone http://llvm.org/git/libcxxabi.git
$ cd ~/src/llvm/tools
$ git clone http://llvm.org/git/clang.git
$ cd ~/src/llvm/tools/clang/tools
$ git clone http://llvm.org/git/clang-tools-extra.git extra
```

Next, do the build:

```bash
$ mkdir -p ~/src/llvm/build
$ cd ~/src/llvm/build
$ cmake -DCMAKE_INSTALL_PREFIX=/opt/clang -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
$ make -j$(getconf _NPROCESSORS_ONLN)
$ sudo make install
```

On OS X you should also pass ``-DDEFAULT_SYSROOT=$(xcrun -show-sdk-path)``
to the cmake command. I could not find documentation for this, but
it appears to default the ``-isysroot`` option. If you don't specify
this here, you will need to pass ``-isysroot`` and ``-Wl,-syslibroot``
everywhere.

Also, read the LLVM [build](http://llvm.org/docs/CMake.html) and the
[git mirror](http://llvm.org/docs/GettingStarted.html#git-mirror)
documentation.
