---
title: "Tracing rmdir system calls with SystemTap"
date: 2017-09-25T12:54:42+10:00
tags: [linux,systemtap,debugging]
featured_image: ""
description: ""
---

I wanted to know who was removing the [Mesos](https://mesos.apache.org/)
memory cgroups hierarchy and why, so I turned to
[SystemTap](https://sourceware.org/systemtap/).
Here's my one-liner:

    sudo stap \
    -d /usr/lib/systemd/libsystemd-shared-233.so \
    -d /usr/lib64/libc-2.25.so \
    -d /usr/lib/systemd/systemd \
    -e 'probe kernel.function("sys_rmdir") {
      printf("%s(%s): %s\n", execname(), pp(), user_string($pathname));
      print_ubacktrace();
    }'

Note that you have to feed in the binaries you expect to see in order to get user stack traces.

The corresponding `systemd` stack trace was:

    systemd(kernel.function("SyS_rmdir@fs/namei.c:3936")): /sys/fs/cgroup/memory
     0x7f7559be3c47 : rmdir+0x7/0x30 [/usr/lib64/libc-2.25.so]
     0x7f755b2fa169 : cg_trim+0x109/0x1f0     [/usr/lib/systemd/libsystemd-shared-233.so]
     0x7f755b2fc280 : cg_create_everywhere+0xa0/0xb0 [/usr/lib/systemd/libsystemd-shared-233.so]
     0x55d79d8bf861 : unit_realize_cgroup_now.lto_priv.582+0x101/0x23b0 [/usr/lib/systemd/systemd]
     0x55d79d8bfc88 : unit_realize_cgroup_now.lto_priv.582+0x528/0x23b0 [/usr/lib/systemd/systemd]
     0x55d79d8bfc88 : unit_realize_cgroup_now.lto_priv.582+0x528/0x23b0 [/usr/lib/systemd/systemd]
     0x55d79d8c1ce2 : unit_realize_cgroup+0x1d2/0x200 [/usr/lib/systemd/systemd]
     0x55d79d8a1daa : slice_start.lto_priv.202+0x2a/0x90 [/usr/lib/systemd/systemd]
     0x55d79d8b898c : job_perform_on_unit.lto_priv.583+0x5fc/0x6d0 [/usr/lib/systemd/systemd]
     0x55d79d85e3a8 : manager_dispatch_run_queue+0x258/0x640 [/usr/lib/systemd/systemd]
     0x7f755b33f8ca : source_dispatch+0x14a/0x380 [/usr/lib/systemd/libsystemd-shared-233.so]
     0x7f755b33fbca : sd_event_dispatch+0xca/0x1d0 [/usr/lib/systemd/libsystemd-shared-233.so]
     0x7f755b341007 : sd_event_run+0x77/0x200 [/usr/lib/systemd/libsystemd-shared-233.so]
     0x55d79d853384 : manager_loop+0x605/0x676 [/usr/lib/systemd/systemd]
