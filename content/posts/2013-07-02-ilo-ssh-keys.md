---
title: "iLO SSH key management"
date: 2013-07-02T15:49:11+10:00
tags: [ilo,ssh]
featured_image: ""
description: ""
---

A few notes and rants about managing SSH keys with HP’s extremely annoying iLO interface.

1. The iLO ssh console does not support fetching SSH keys over HTTPS. This
prevents you keeping them somewhere useful like github.
1. When you upload a SSH key and it fails, the iLO web interface will
tell you that it needs a PEM-formatted DSA public key. You will find
that ssh-keygen has no way to produce this. Fortunately it is a lie.
1. The SSH key comments field must not contain spaces. iLO accepts the SSH
format, but the comment must be of the form “user@host” or go to (2).
