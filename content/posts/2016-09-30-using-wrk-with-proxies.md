---
title: "Using wrk with proxies"
date: 2016-09-30T13:27:30+10:00
tags: [http,wrk,benchmarking]
featured_image: ""
description: ""
---

Based on
[this](http://atodorov.org/blog/2014/11/18/proxy-support-for-wrk-http-benchmarking-tool/)
extremely helpful post, a slight extension to make it easier to use
[wrk](https://github.com/wg/wrk) with a HTTP proxy.

```lua
url = ''
host = ''

init = function(args)
    url = args[1] -- proxy needs absolute URL

    -- Capture the hostname from the target URL.
    _, _, host = string.find(url, 'http://([^/]+)/')
end

request = function()
    return wrk.format("GET", url, { Host = host })
end
```

Usage is like this:

     $ wrk -s proxy.wrk --connections 1 --threads 1 --duration 10 http://127.0.0.1:8001/ -- http://example.com/foo/bar/baz
