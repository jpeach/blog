---
title: "Creating multiple resources with Salt"
date: 2013-08-05T15:41:19+10:00
tags: [saltstack]
featured_image: ""
description: ""
---

I wanted to create a Salt Stack state that manages multiple directories. I figured that there was a way to do this, but could not see a good example in the documentation. Fortunately, the very helpful #salt IRC channel pointed me to the answer:

```
hierarchy:
   file.directory:
     - user: root
     - group: root
     - mode: 755
     - makedirs: True
     - names:
      - /var/lib/hierarchy
      - /var/lib/hierarchy/a
      - /var/lib/hierarchy/b
      - /var/lib/hierarchy/b/c
      - /var/lib/hierarchy/b/c/d
```
