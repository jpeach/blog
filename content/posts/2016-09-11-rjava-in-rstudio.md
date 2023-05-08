---
title: Loading the “rJava” package into RStudio
date: 2016-09-11T13:30:50+10:00
tags: []
featured_image: ""
description: ""
---

Poking around the interwebs, everyone seems to get into trouble loading
the ``rJava`` package into [RStudio](https://www.rstudio.com). It seems
like there are enough people trying to make this work that it should
just work out of the box, but then again, what do I know?

Here's what worked for me:

```R
jdk <- system2("/usr/libexec/java_home", stdout=TRUE)
dyn.load(paste(jdk, "jre/lib/server/libjvm.dylib", sep="/"))
library("rJava")
```

This manually figures out where libjvm.dylib is and loads it prior to opening the rJava library. Ick.
