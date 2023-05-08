---
title: "Per-module logging with glog"
date: 2015-07-09T15:22:15+10:00
tags: []
featured_image: ""
description: ""
---

I spent a few hours trying to implement per-module logging in the
[Mesos](http://mesos.apache.org) logging toggle. The [Google
logging library](https://github.com/google/glog) supports the
`--vmodule` flags to toggle the logging level on a per-module
basis, which looked promising. You can set this at run time using the
[SelVLOGLevel](https://github.com/google/glog/blob/master/src/glog/vlog_is_on.h.in#L104)
API.

Unfortunately, the implementation of the
[VLOG_IS_ON](https://github.com/google/glog/blob/master/src/glog/vlog_is_on.h.in#L82)
macro is such that you need to set a per-module log level before the
logging call site is hit for the first time, so this is clearly intended
only for startup. You can't dial up logging on  arbitrary logging modules
at any time.
