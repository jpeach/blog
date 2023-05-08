---
title: "Ccache and Bazel"
date: 2019-09-05T19:18:22+10:00
tags: [ccache,bazel]
featured_image: ""
description: ""
---

Bazel defaults to building code in a sandbox that remounts most of the
filesystem read-only. This means that if you are using ccache (Fedora,
for example, will enable it by creating appropriate symlinks when you
install the package) the compile job will fail because it can't write
to the cache directory.

The straightforward fix to this is to create a
[bazelrc](https://docs.bazel.build/versions/master/guide.html#bazelrc)
file which specifies the Bazel
[sandbox_writable_path](https://docs.bazel.build/versions/master/command-line-reference.html#flag--sandbox_writable_path)
flag to make the cache directory writeable.

```
$ cat ~/.bazelrc
build --sandbox_writable_path=/home/jpeach/.ccache
```

Is ccache actually worth it with Bazel? Good question!
