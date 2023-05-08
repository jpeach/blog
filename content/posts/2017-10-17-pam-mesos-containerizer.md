---
title: "PAM support in the Mesos containerizer"
date: 2017-10-17T12:50:53+10:00
tags: [pam,mesos,containers]
featured_image: ""
description: ""
---

Recently, it occurred to me that running a containerized
task is concentually very similar to having a remote
session on an anonymous compute agent. The traditional way
for operators to influence (i.e. configure, control, log)
the environment of a remote user session is by the use of
[PAM](https://en.wikipedia.org/wiki/Pluggable_authentication_module)
modules. One of the applications that I had in mind was the use of the
[pam_loginuid](http://man7.org/linux/man-pages/man8/pam_loginuid.8.html)
module to set the linux audit ID so that containers audit events can be
attributed to the task user rather than to the container orchestrator. In
describing this idea to others, I was a little surprised to find that
PAM was quite unfamiliar to a lot of people. Even though it is such
a fundamental part of every UNIX-like system, it seems to be quite an
esoteric subsystem these days.

Anyway, I wrote a [design
document](https://docs.google.com/document/d/1nlWC7ArgQRr5f_uH5wJ0AGV8BHMhoKgYf__RTmu7TKk)
describing how I think this would work with the Mesos containerizer and
we discussed it in the Mesos Containerization Working Group which was
recorded on [YouTube](https://youtu.be/0JhCDckQh-g). The Mesos integration
seems a little tricky, but quite do-able, and I think this would have some
fun applications. Hopefully I will get time to actually implement it :)
