---
title: "Cannot extend ID. It is not part of highstate"
date: 2013-09-19T15:39:12+10:00
tags: [saltstack]
featured_image: ""
description: ""
---

I spent quite a while scratching my head over the following error message from [Salt](http://saltstack.com):

    Cannot extend ID trafficserver in "base:trafficserver.collector". It is not part of the high state.

This actually means that you used a requisite clause like ``watch_in`` to inject a dependency into a state that Salt cannot resolve. I filed bug [7336](https://github.com/saltstack/salt/issues/7336).
