---
title: "Using Address Sanitizer with TrafficServer"
date: 2016-11-04T13:21:54+10:00
tags: [trafficserver,debugging]
featured_image: ""
description: ""
---

Verifying [Traffic Server](https://trafficserver.apache.org) with
[AddressSanitizer](https://github.com/google/sanitizers) is fairly
straight-forward. On Linux, you need recent `gcc` or `clang` and the
`libasan` library. On OS X, `libasan` wasn't present, so I just switched
to Linux ;)

You should give `--enable-asan` to `configure` when you build. The build
system will enable ASAN on all the parts that should have it. Then,
whenever you run you will get ASAN checking memory state.

[LeakSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer)
reports leaks from an
[atexit(3)](http://man7.org/linux/man-pages/man3/atexit.3.html)
handler, so you need to ensure that the program exits rather than
calls [_exit(2)](http://man7.org/linux/man-pages/man2/_exit.2.html) or
dumps core. If you are using `traffic_manager`, then you can set the
[`PROXY_AUTO_EXIT`](https://docs.trafficserver.apache.org/appendices/command-line/traffic_server.en.html#envvar-PROXY_AUTO_EXIT)
variable to have `traffic_server` voluntarily exit
after the specified number of seconds. LeakSanitizer will kindly print
a leak report each time it exits, and `traffic_manager` will start it
up for you to test again.
