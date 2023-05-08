---
title: "Updating go_resources in Homebrew"
date: 2016-06-27T15:13:02+10:00
tags: [go,homebrew]
featured_image: ""
description: ""
---

This is a quick note to myself about how to update the ``go_resources`` in a [Homebrew](http://brew.sh) formula.

First, install ``godep`` and the Homebrew dev tools:

    $ cd $GOPATH
    $ go get -u github.com/tools/godep
    $ brew tap homebrew/dev-tools

Next, generate a ``Godeps`` file in your Go project

    $ cd $GOPATH/src/github.com/me/my-project
    $ $GOPATH/bin/godep save .
    
Now you can get ``brew`` to generated the ``go_resources`` that you can just paste into your formula:

    $ cd $GOPATH/src/github.com/me/my-project
    $ brew go-resources
