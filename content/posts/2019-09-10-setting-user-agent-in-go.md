---
title: "Setting the HTTP User Agent in Go"
date: 2019-09-10T07:04:50+10:00
tags: [go,http]
featured_image: ""
description: ""
---

Here’s the smallest amount of code I could come up with to set the
user agent when making a HTTP request in Go:

```Go
type UserAgent string

func (u UserAgent) RoundTrip(r *http.Request) (*http.Response, error) {
    r.Header.Set("User-Agent", string(u))
    return http.DefaultTransport.RoundTrip(r)
}

http.DefaultClient.Transport = UserAgent("my-great-program")
```

Note that this isn't really legal, since the
[RoundTripper](https://pkg.go.dev/net/http#RoundTripper)
is not supposed to modify the request.
