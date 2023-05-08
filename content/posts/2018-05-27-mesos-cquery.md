---
title: "Mesos config for cquery"
date: 2018-05-27T19:21:58+10:00
tags: [mesos,cquery,lsp]
featured_image: ""
description: ""
---

The canonical way to feed your project to cquery is to generate a
`compile_commands.json` file, possibly using cmake. Every time I switch
my Mesos work to the cmake build, I live to regret it, either because
the component I'm working on isn't implemented in the cmake build or I
end up wanting to install the build (and that isn't implemented in cmake).

So here's a [.cquery](https://github.com/cquery-project/cquery/wiki/Getting-started#cquery) file that makes cquery work pretty well with Mesos:
```
%clang
%cpp -std=c++11
-Iinclude
-Isrc
-Ibuild/src
-Ibuild/include
-I3rdparty/stout/include
-I3rdparty/libprocess/include
-Ibuild/3rdparty/googletest-release-1.8.0/googletest
-Ibuild/3rdparty/glog-0.3.3/src
-Ibuild/3rdparty/picojson-1.3.0
-DLIBDIR="/lib"
-DVERSION="1.0"
```
