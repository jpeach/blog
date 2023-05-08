---
title: "Python pip HTTPS proxying"
date: 2016-09-17T13:29:10+10:00
tags: [python,https,proxy]
featured_image: ""
description: ""
---

I investigated an issue where ``pip`` fails to establish a TLS tunnel
through a HTTP proxy because the proxy response with 400 (Bad Request).
 
It turns out tht ``pip`` sends this ``CONNECT`` request:
 
     CONNECT pypi.python.org:443 HTTP/1.0

 Now, HTTP/1.1 requires a ``Host`` header, so 400 would be the correct
 response in that case. ``CONNECT`` wasn't defined in the original
 HTTP/1.0 [RFC 1945](https://tools.ietf.org/html/rfc1945),
 but Bryan Call pointed me to the
 [draft-luotonen-web-proxy-tunneling](https://tools.ietf.org/html/draft-luotonen-web-proxy-tunneling-01)
 so I guess at one point this was a thing.
