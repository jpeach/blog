---
title: "GNU make dependency generation"
date: 2015-11-02T15:18:27+10:00
tags: [make]
featured_image: ""
description: ""
---

Although I would normally use [automake](https://autotools.io/), recently I needed to write a Makefile by hand, so I went down the path of figuring out how to get gcc to generate dependency files as a side-effect of compilation:

```make
# Build rule for compiling C++ with dependecy generation as a side-effect. The
# dependencies go into a .deps directory at the same level as the source file.
%.o: %.cpp
    @$(MKDIR) $(*D)/.deps
    $(CXX) $(CXXFLAGS) $(CPPFLAGS) -MP -MF $(*D)/.deps/$(*F).d -MMD -c $
```
