---
title: "Using cquery from vim with cscope-lsp"
date: 2018-05-02T19:24:42+10:00
tags: [cscope,cquery,vim,lsp]
featured_image: ""
description: ""
---

A while ago I tried [cquery](https://github.com/cquery-project/cquery)
with Vim, but was a bit unsatisfied with the integration. I could probably
have altered my key bindings to perform LSP searches instead of cscope
searches, but I also really quite like the integrated tags stack you
get with the built-in Vim cscope support.

So I thought that it couldn't be too hard to implement the cscope
line protocol with a cquery backend, and it turns out that it
wasn't. I wrote [cscope-lsp](https://github.com/jpeach/cscope-lsp)
on and off during the week and it's almost at a useful point. Most
search types work, and vim will drive it without too much
configuration. The main missing feature is that I need to resolve a
[LSP](https://microsoft.github.io/language-server-protocol/specification)
Location to an actual text string for the cscope results to be meaningful,
but in many cases, cscope-lsp just jumps directly to the result you want,
which is a big improvement for my workflow.
