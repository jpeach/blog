---
title: "Leak detection with tcmalloc"
date: 2015-01-14T15:29:43+10:00
tags: [linux,malloc,tcmalloc]
featured_image: ""
description: ""
---

[tcmalloc](https://code.google.com/p/gperftools/?redir=1)
has a built-in leak detection mechanism. It took me a couple
of tries to figure out how to work it, even after reading the
[documentation](http://google-perftools.googlecode.com/svn/trunk/doc/heap_checker.html).
At least on Centos 7, the trick is to make sure you install the pprof
package as well as gperftools-libs package. You will also need to set
the `PPROF_PATH` environment variable so that the tcmalloc runtime can
find proof. If you don't do this, then the leaks report will not resolve
symbols, so the stack traces will not be that useful.

Note that by default, the leak report is emitted on exit, so it if your
program calls `_exit()` or aborts, you won't get any report. If you are
trying to check [TrafficServer](https://trafficserver.apache.org/),
it's worth building with `--disable-freelist` to avoid false positives
and verify any possible freelist leaks.
