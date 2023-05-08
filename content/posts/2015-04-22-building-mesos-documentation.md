---
title: "Building the Mesos documentation"
date: 2015-04-22T15:25:24+10:00
tags: [mesos,documentation]
featured_image: ""
description: ""
---

I made a minor change to the documentation and I wanted to test it, so I had to figure out how to build the [Mesos](http://mesos.apache.org/) documentation. I’m doing this in a CentOS 7 VM, but I guess something similar would work on different platforms.

First, you need to know that Mesos uses [Middleman](http://middlemanapp.com) to build the website from a set of markdown files. Next, you need to know that while the documentation itself is in the main Mesos repository, the Middleman configuration and site tooling is in a separate SVN repository.

So, get started by checking out both repositories:
```
[vagrant@centos-7 src]$ git clone https://github.com/apache/mesos.git
...
[vagrant@centos-7 src]$ svn co http://svn.apache.org/repos/asf/mesos/site/ mesos-site
```

Now you can build the documentation from the mesos-site repository following the instructions in the ``README.md`` you will find there:

```
[vagrant@centos-7 src]$ cd mesos-site
[vagrant@centos-7 mesos-site]$ gem install bundler
...
[vagrant@centos-7 mesos-site]$ gem bundle install
...
[vagrant@centos-7 mesos-site]$ bundle install
```

I also had to edit the ``Gemfile`` to add ``rake`` since I didn't have it installed already for some reason. Once all the dependencies are installed, you can build and test the whole website:

```
[vagrant@centos-7 mesos-site]$ bundle rake
...
[vagrant@centos-7 mesos-site]$ bundle middleman server
== The Middleman is loading
== LiveReload is waiting for a browser to connect
== The Middleman is standing watch at http://0.0.0.0:4567
== Inspect your site configuration at http://0.0.0.0:4567/__middleman/
...
```

At this point you have the upstream documentation built, but the goal of
this exercise was to test your own documentatino changes, so we have to
point the documentation build to our local changes. I didn\'t see a way
to easily do this, so I hacked it into the `Rakefile`:

```
[vagrant@centos-7 mesos-site]svn diff Rakefile
Index: Rakefile
===================================================================
--- Rakefile	(revision 1675427)
+++ Rakefile	(working copy)
@@ -27,8 +27,8 @@
   if File.exists?(mesos_dir)
     system("pushd #{mesos_dir} && git pull origin master && popd")
   else
-    FileUtils.mkdir_p(mesos_dir)
-    system("git clone --depth 1 http://git-wip-us.apache.org/repos/asf/mesos.git #{mesos_dir}")
+    FileUtils.mkdir_p(File.dirname(mesos_dir))
+    FileUtils.ln_s("/opt/home/src/mesos.git", mesos_dir)
   end
 end
```

All we are doing here is symlinking to our local copy of the Mesos
documentation rather than pulling it down from the upstream git
repository. Once you rebuild and start the Middleman server you will be
able to browse the full site and verify your changes.
