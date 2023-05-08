---
title: "Argument parsing in Traffic Server plugins"
date: 2012-12-18T15:57:55+10:00
tags: [trafficserver]
featured_image: ""
description: ""
---

When you write a new [Traffic Server](http://trafficserver.apache.org)
plugin, you have to choose whether to write a remap plugin, a global
plugin or both. There are different plugin entry points for global and
remap plugins and you will find yourself having to parse command-line
argument from two different entry points:

```
tsapi void TSPluginInit(int argc, const char* argv[]);
tsapi TSReturnCode TSRemapNewInstance(int argc, char* argv[], void** ih, char* errbuf, int errbuf_size);
```

Since we are parsing command-line options, it makes sense to use `getopt`
or `getopt_long` to do the parsing. I prefer to use `getopt_long`
because I find long options more legible and memorable.

However, `TSPluginInit` and `TSRemapNewInstance` do not pass the same
argument vector, so if you are unwary you will process arguments
incorrectly. The `TSPluginInit` argument vector is the plugin name
followed by the arguments, whereas the `TSRemapNewInstance` argument
vector contains the to and from URLs from the remap rule, followed by the
remaining arguments. Since `getopt_long` expects the first argument to
be the program name, it's ok with the `TSPluginInit` argument vector,
but you need to massage the `TSRemapNewInstance` one. I usually just
increment it, because it really doesn't matter what you pass in the
first entry, just as long as there is one.

The final trap is to remember that `optind` is a global variable and you
have to reset it before you call `getopt_long` for the second time. Since
it's harmless to reset it each time, my code just always sets it. For
a plugin that supports both global and remap modes, you should end up
with something like this:

```C
optind = 0;
for (;;) {
    int opt;
    
    opt = getopt_long(argc, (char * const *)argv, "", longopt, NULL);
    switch (opt) {
    /* TODO: check flags */
    default:
        snprintf(errbuf, errbuf_size, "invalid option '%d'", opt);
        return TS_ERROR;
    }
    
    if (opt == -1) {
        break;
    }
}
```
