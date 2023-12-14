---
title: "Git LFS With Homebrew"
date: 2023-12-14T15:23:36+11:00
tags: [git,homebrew]
---

So, I've been working on an internal [Homebrew](https://brew.sh)
tap, and have been trying to get the automatic bottle packaging
actions working. There's a good
[blog post](https://brew.sh/2020/11/18/homebrew-tap-with-bottles-uploaded-to-github-releases/)
that describes the outcomes, but the Homebrew project supports
internal taps on a best effort basis, so this doesn't dig into all
the issues you might encounter when setting it up internally.

By far the most time consuming issue I had was getting the git clone to find
`git-lfs`. One of the formulae I was building had a
[gitattributes](https://git-scm.com/docs/gitattributes) file that tags some
paths for `git-lfs`. When Homebrew tries to clone the repository, you get
the dreaded `git-lfs filter-process: git-lfs: command not found` error message,
and the clone fails.
This happens on macOS builders because Homebrew (for good reasons)
forces the PATH to be `"/usr/bin:/bin:/usr/sbin:/sbin"`. Since macOS
doesn't install `git-lfs` by default it would clearly not be found.

I tried a lot of different ways to work around this. The basic approch I took
was using Homebrew itself to install `git-lfs` on the Github runner and trying
to convince the `git` command to use it. You can't just symlink `git-lfs` into
`/usr/bin` because that is a readonly part of the filesystem due to
[SIP](https://support.apple.com/en-au/102149). I thought that setting the
`HOMEBREW_FORCE_BREWED_GIT` environment variable to get Homebrew to use it's
own version of git would work, but that didn't so much that I could see.
I also tried setting up a global `~/.gitconfig` with a `git-lfs`
filter configuration passing absolute paths to the `git-lfs` command.
This almost works, but when `git-lfs` runs it installs `post-checkout`
hooks as a side-effect, and those
[hooks](https://github.com/git-lfs/git-lfs/blob/main/lfs/hook.go)
require `git-lfs` to be in the path.

So in the end, the solution is:
```
sudo ln -sf $(brew --prefix git-lfs)/bin/git-lfs $(/usr/bin/git --exec-path)/git-lfs
```

This symlinks the Homebrew version of `git-lfs` into the exec path
of the system `git` command. I can't say that I know enough about
git internals to say exactly why this works, but it does, and it
cost me way more time that it was worth to figure it out.

