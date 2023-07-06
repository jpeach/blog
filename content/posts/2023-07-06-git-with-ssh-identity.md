---
title: "Using a Specifc SSH Indentity with Git"
date: 2023-07-06T12:35:31+10:00
tags: [git,ssh]
---

Normally, it's common to configure multiple SSH keys git Git access,
but there are situations where you need to use a specific key. In this
case, you don't want to let ssh just choose the first working key,
you want it to use a specific SSH identity. The use case that I had
was that for a certain set of source repositories, I wanted to use a
particular key because that specific key was approved by a particular
GitHub organization.

The first part of the puzzle is a SSH simple wrapper script that forces the
spacific key:

```bash
#! /usr/bin/env bash

exec ssh -F none \
    -o "IdentitiesOnly yes" \
    -o "IdentityFile ~/.ssh/organization/id_ed25519" \
    "$@"

```

Here, I've used the `-F none` argument to turn off reading the default
ssh config file. This works for me because I basically have a trivial
ssh configration. Then I turn off the ssh-agent identities and add just
the identity that I want.

The second part is to use direnv to set up the git to use the wrapper script.

```
$ pwd
/Users/jpeach/organization

$ ls -Fal
total 8
drwxr-xr-x   5 jpeach staff  160 Jul  6 12:26 ./
drwxr-xr-x+ 81 jpeach staff 2592 Jul  6 12:26 ../
-rw-r--r--   1 jpeach staff   26 Jul  6 12:15 .envrc
-rwxr-xr-x   1 jpeach staff  131 Jul  6 12:26 ssh*

$  cat .envrc
export GIT_SSH=$(pwd)/ssh
```

So here, we have a top-level directory that will contain all the
checked out repositories for the organization. We also have the
trivial ssh wrapper script, and a direnv file that sets the `$GIT_SSH`
environment variable. The effect of all this is that when I change into
the organization directory, git will automatically use the ssh wrapper
and I don't need to remember to set anything else up. Note that direnv
evaluates the `.envrc` file in the organization directory (so that
`$(pwd)` gives the expected result), even if you change straight into
a subdirectory.

