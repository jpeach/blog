---
title: "TrafficServer GET request walkthrough"
date: 2012-12-18T15:51:33+10:00
tags: [trafficserver]
featured_image: ""
description: ""
---

[Traffic Server](http://trafficserver.apache.org) request processing can be a little complex, with multiple state machines working at the same time and a lot of objects interacting in complex ways, so I thought it would be fun to reverse engineer the code flow from a log trace. I guess that it wasn't as much fun as I had hoped, but it was educational.

This is a [GET](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.3) request with a [Range](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.35) header. The requested document is not currently cached, so [Traffic Server](http://trafficserver.apache.org) just proxies the request. This is the simplest case since we don't hit the cache and we don't store the response in the cache.

Here we go ...

```
[Dec 13 08:58:38.738] DEBUG: (http_seq) [HttpAccept:mainEvent 0x7fc41b814820] accepted connection from 127.0.0.1:49650 transport type = 0
```

We received `NET_EVENT_ACCEPT` on a listen socket. We create a `HttpClientSession` object to represent the accepted connection. We call `HttpClientSession::new_connection`to start up the session.

```
[Dec 13 08:58:38.738] DEBUG: (http_cs) [6577] session born, netvc 0x7fc41b814820
```

We are now in `HttpClientSession::new_connection` and we do some startup tasks and invoke the `TS_HTTP_SSN_START_HOOK`. Upon return, we will end up in `HttpClientSession::handle_api_return` and call `HttpClientSession::new_transaction`.

```
[Dec 13 08:58:38.738] DEBUG: (http_cs) [6577] using accept inactivity timeout [120 seconds]
```

We are in `HttpClientSession::new_transaction`. We create a HttpSM using `HttpSM::allocate` and attach it as `HttpClientSession::current_reader`.

```
[Dec 13 08:58:38.738] DEBUG: (http_cs) [6577] Starting transaction 1 using sm [6652]
```

We start the `HttpSM` state machine by calling `HttpSM::attach_client_session`. The `HttpSM` gets a pointer back to the `IOBufferReader` (`HttpClientSession::sm_reader`) and starts reading the data by calling `HttpClientSession::do_io_read`. We set `HttpSM::vc_handler` to the next state handler. We now invoke the `TS_HTTP_TXN_START_HOOK` via the sequence `state_add_to_list` → `do_api_callout` → `do_api_callout_internal`.

```
[Dec 13 08:58:38.738] DEBUG: (http) [6652] [HttpSM::main_handler, VC_EVENT_READ_READY]
```

The `HttpSM`gets a read event.

```
[Dec 13 08:58:38.738] DEBUG: (http) [6652] [&HttpSM::state_read_client_request_header, VC_EVENT_READ_READY]
```

Jump to the state handler, which is `HttpSM::state_read_client_request_header`.

```
[Dec 13 08:58:38.738] DEBUG: (http) [6652] done parsing client request header
```

We successfully parsed the HTTP header, which happened in `HTTPHdr::parse_req`. The actual HTTP protocol parsing is in `http_parser_parse_req`.

```
[Dec 13 08:58:38.738] DEBUG: (http_trans) START HttpTransact::ModifyRequest
```

`HttpSM::call_transact_and_set_next_state` sends us into `HttpTransact::ModifyRequest`. From here, there are 2 alternating state machines, in `HttpSM` and `HttpTransact`. The action is getting a little hard to follow ...

```
[Dec 13 08:58:38.738] DEBUG: (http_trans) [ink_cluster_time] local: 1355417918, highest_delta: 0, cluster: 1355417918
[Dec 13 08:58:38.738] DEBUG: (http_trans) END HttpTransact::ModifyRequest
```

`HttpTransact::ModifyRequest` is done. The point of that was to extract and normalize information we will need to process the request. The `TRANSACT_RETURN` takes us into `HttpTransact::StartRemapRequest`.

```
[Dec 13 08:58:38.738] DEBUG: (http_trans) Next action HTTP_API_READ_REQUEST_HDR; HttpTransact::StartRemapRequest
[Dec 13 08:58:38.738] DEBUG: (http) [6652] State Transition: STATE_UNDEFINED -&gt; API_READ_REQUEST_HDR
[Dec 13 08:58:38.738] DEBUG: (http_trans) START HttpTransact::StartRemapRequest
```

Get set up for remapping, but before we do, dump the parsed HTTP request ...

```
[Dec 13 08:58:38.738] DEBUG: (http_trans) Before Remapping:
[Dec 13 08:58:38.738] DEBUG: (http) HTTP_HEADER 0x113abc088: [T: 3, L:   48, OBJFLAGS: 0]  
[Dec 13 08:58:38.738] DEBUG: (http) [TYPE: REQ, V: 10001, URL: 0x113abc308, METHOD: "GET", METHOD_LEN: 3, FIELDS: 0x113abc0b8]
[Dec 13 08:58:38.738] DEBUG: (http) URL 0x113abc308: [T: 2, L:  112, OBJFLAGS: 0]  
[Dec 13 08:58:38.738] DEBUG: (http) [URLTYPE: 1, SWKSIDX: 94,
[Dec 13 08:58:38.738] DEBUG: (http) 	SCHEME: "http", SCHEME_LEN: 4,
[Dec 13 08:58:38.738] DEBUG: (http) 	USER: "", USER_LEN: 0,
[Dec 13 08:58:38.738] DEBUG: (http) 	PASSWORD: "", PASSWORD_LEN: 0,
[Dec 13 08:58:38.738] DEBUG: (http) 	HOST: "", HOST_LEN: 0,
[Dec 13 08:58:38.738] DEBUG: (http) 	PORT: "", PORT_LEN: 0, PORT_NUM: 0
[Dec 13 08:58:38.738] DEBUG: (http) 	PATH: "Telnet.dmg", PATH_LEN: 10,
[Dec 13 08:58:38.738] DEBUG: (http) 	PARAMS: "", PARAMS_LEN: 0,
[Dec 13 08:58:38.738] DEBUG: (http) 	QUERY: "", QUERY_LEN: 0,
[Dec 13 08:58:38.738] DEBUG: (http) 	FRAGMENT: "", FRAGMENT_LEN: 0]
[Dec 13 08:58:38.738] DEBUG: (http) MIME_HEADER 0x113abc0b8: [T: 4, L:  592, OBJFLAGS: 0]  
[Dec 13 08:58:38.738] DEBUG: (http) 
	[PBITS: 0x0008040001000001, SLACC: 0xFFFFFFF2FFFFFFFFFFFFFFFFFFF0FFF3, HEADBLK: 0x113abc0f8, TAILBLK: 0x113abc0f8]
[Dec 13 08:58:38.738] DEBUG: (http) 	[CBITS: 0x00000000, T_MAXAGE: 0, T_SMAXAGE: 0, T_MAXSTALE: 0, T_MINFRESH: 0, PNO$: 0]
[Dec 13 08:58:38.738] DEBUG: (http) FIELD_BLOCK 0x113abc0f8: [T: 5, L:  528, OBJFLAGS: 0]  
[Dec 13 08:58:38.738] DEBUG: (http) [FREETOP: 4, NEXTBLK: 0x0]
[Dec 13 08:58:38.738] DEBUG: (http) 	SLOT # 0 (0x113abc108), LIVE    
[Dec 13 08:58:38.738] DEBUG: (http) [N: "User-Agent", N_LEN: 10, N_IDX: 64, 
[Dec 13 08:58:38.738] DEBUG: (http) V: "curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5", V_LEN: 78, 
[Dec 13 08:58:38.738] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 92, F: 1]
[Dec 13 08:58:38.738] DEBUG: (http) 
[Dec 13 08:58:38.738] DEBUG: (http) 	SLOT # 1 (0x113abc128), LIVE    
[Dec 13 08:58:38.738] DEBUG: (http) [N: "Host", N_LEN: 4, N_IDX: 30, 
[Dec 13 08:58:38.738] DEBUG: (http) V: "localhost", V_LEN: 9, 
[Dec 13 08:58:38.738] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
[Dec 13 08:58:38.738] DEBUG: (http) 
[Dec 13 08:58:38.738] DEBUG: (http) 	SLOT # 2 (0x113abc148), LIVE    
[Dec 13 08:58:38.738] DEBUG: (http) [N: "Accept", N_LEN: 6, N_IDX: 4, 
[Dec 13 08:58:38.738] DEBUG: (http) V: "*/*", V_LEN: 3, 
[Dec 13 08:58:38.738] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 13, F: 1]
[Dec 13 08:58:38.738] DEBUG: (http) 
[Dec 13 08:58:38.738] DEBUG: (http) 	SLOT # 3 (0x113abc168), LIVE    
[Dec 13 08:58:38.738] DEBUG: (http) [N: "Range", N_LEN: 5, N_IDX: 52, 
[Dec 13 08:58:38.738] DEBUG: (http) V: "bytes=-1", V_LEN: 8, 
[Dec 13 08:58:38.738] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
[Dec 13 08:58:38.738] DEBUG: (http) 
[Dec 13 08:58:38.738] DEBUG: (http_trans) END HttpTransact::StartRemapRequest
[Dec 13 08:58:38.738] DEBUG: (http_trans) Next action HTTP_API_PRE_REMAP; HttpTransact::PerformRemap
[Dec 13 08:58:38.738] DEBUG: (http) [6652] State Transition: API_READ_REQUEST_HDR -&gt; HTTP_API_PRE_REMAP
[Dec 13 08:58:38.738] DEBUG: (http_trans) Inside PerformRemap
```

The `HttpTransact::PerformRemap` state doesn't actually do anything; it's a trampoline state to set `HttpTransact::next_action` to `HttpTransact::HTTP_REMAP_REQUEST`. This cause `HttpSM::set_next_state`to invoke the remapping engine.

```
[Dec 13 08:58:38.738] DEBUG: (http_trans) Next action HTTP_REMAP_REQUEST; HttpTransact::EndRemapRequest
[Dec 13 08:58:38.738] DEBUG: (http) [6652] State Transition: HTTP_API_PRE_REMAP -&gt; HTTP_REMAP_REQUEST
[Dec 13 08:58:38.738] DEBUG: (http_seq) [HttpSM::do_remap_request] Remapping request
[Dec 13 08:58:38.738] DEBUG: (http_trans) START HttpTransact::EndRemapRequest
[Dec 13 08:58:38.738] DEBUG: (http_trans) EndRemapRequest host is 10.10.10.9
```

We finish remapping and tidy up some loose ends. Dump the remapped transaction for debugging purposes and call out to the `TS_HTTP_POST_REMAP_HOOK`.

```
[Dec 13 08:58:38.739] DEBUG: (http_trans) After Remapping:
[Dec 13 08:58:38.739] DEBUG: (http) HTTP_HEADER 0x113abc088: [T: 3, L:   48, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [TYPE: REQ, V: 10001, URL: 0x113abc308, METHOD: "GET", METHOD_LEN: 3, FIELDS: 0x113abc0b8]
[Dec 13 08:58:38.739] DEBUG: (http) URL 0x113abc308: [T: 2, L:  112, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [URLTYPE: 1, SWKSIDX: 94,
[Dec 13 08:58:38.739] DEBUG: (http) 	SCHEME: "http", SCHEME_LEN: 4,
[Dec 13 08:58:38.739] DEBUG: (http) 	USER: "", USER_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	PASSWORD: "", PASSWORD_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	HOST: "10.10.10.9", HOST_LEN: 13,
[Dec 13 08:58:38.739] DEBUG: (http) 	PORT: "", PORT_LEN: 0, PORT_NUM: 0
[Dec 13 08:58:38.739] DEBUG: (http) 	PATH: "DiskStoragePath/Telnet.dmg", PATH_LEN: 26,
[Dec 13 08:58:38.739] DEBUG: (http) 	PARAMS: "", PARAMS_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	QUERY: "", QUERY_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	FRAGMENT: "", FRAGMENT_LEN: 0]
[Dec 13 08:58:38.739] DEBUG: (http) MIME_HEADER 0x113abc0b8: [T: 4, L:  592, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) 
	[PBITS: 0x0008040001000001, SLACC: 0xFFFFFFF2FFFFFFFFFFFFFFFFFFF0FFF3, HEADBLK: 0x113abc0f8, TAILBLK: 0x113abc0f8]
[Dec 13 08:58:38.739] DEBUG: (http) 	[CBITS: 0x00000000, T_MAXAGE: 0, T_SMAXAGE: 0, T_MAXSTALE: 0, T_MINFRESH: 0, PNO$: 0]
[Dec 13 08:58:38.739] DEBUG: (http) FIELD_BLOCK 0x113abc0f8: [T: 5, L:  528, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [FREETOP: 4, NEXTBLK: 0x0]
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 0 (0x113abc108), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "User-Agent", N_LEN: 10, N_IDX: 64, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5", V_LEN: 78, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 92, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 1 (0x113abc128), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Host", N_LEN: 4, N_IDX: 30, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "localhost", V_LEN: 9, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 2 (0x113abc148), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Accept", N_LEN: 6, N_IDX: 4, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "*/*", V_LEN: 3, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 13, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 3 (0x113abc168), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Range", N_LEN: 5, N_IDX: 52, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "bytes=-1", V_LEN: 8, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http_trans) END HttpTransact::EndRemapRequest
[Dec 13 08:58:38.739] DEBUG: (http_trans) Next action HTTP_API_POST_REMAP; HttpTransact::HandleRequest
[Dec 13 08:58:38.739] DEBUG: (http) [6652] State Transition: HTTP_REMAP_REQUEST -&gt; HTTP_API_POST_REMAP
```

Perform the `TS_HTTP_POST_REMAP_HOOK` callout.

```
[Dec 13 08:58:38.739] DEBUG: (http_trans) START HttpTransact::HandleRequest
[Dec 13 08:58:38.739] DEBUG: (http_trans) [is_request_valid]no request header errors
[Dec 13 08:58:38.739] DEBUG: (http_seq) [HttpTransact::HandleRequest] request valid.
```

We are in `HttpTransact::HandleRequest`, (transitioned from `HttpTransact::EndRemapRequest`). Dump the request state one more time.

```
[Dec 13 08:58:38.739] DEBUG: (http) HTTP_HEADER 0x113abc088: [T: 3, L:   48, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [TYPE: REQ, V: 10001, URL: 0x113abc308, METHOD: "GET", METHOD_LEN: 3, FIELDS: 0x113abc0b8]
[Dec 13 08:58:38.739] DEBUG: (http) URL 0x113abc308: [T: 2, L:  112, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [URLTYPE: 1, SWKSIDX: 94,
[Dec 13 08:58:38.739] DEBUG: (http) 	SCHEME: "http", SCHEME_LEN: 4,
[Dec 13 08:58:38.739] DEBUG: (http) 	USER: "", USER_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	PASSWORD: "", PASSWORD_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	HOST: "10.10.10.9", HOST_LEN: 13,
[Dec 13 08:58:38.739] DEBUG: (http) 	PORT: "", PORT_LEN: 0, PORT_NUM: 0
[Dec 13 08:58:38.739] DEBUG: (http) 	PATH: "DiskStoragePath/Telnet.dmg", PATH_LEN: 26,
[Dec 13 08:58:38.739] DEBUG: (http) 	PARAMS: "", PARAMS_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	QUERY: "", QUERY_LEN: 0,
[Dec 13 08:58:38.739] DEBUG: (http) 	FRAGMENT: "", FRAGMENT_LEN: 0]
[Dec 13 08:58:38.739] DEBUG: (http) MIME_HEADER 0x113abc0b8: [T: 4, L:  592, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) 
	[PBITS: 0x0008040001000001, SLACC: 0xFFFFFFF2FFFFFFFFFFFFFFFFFFF0FFF3, HEADBLK: 0x113abc0f8, TAILBLK: 0x113abc0f8]
[Dec 13 08:58:38.739] DEBUG: (http) 	[CBITS: 0x00000000, T_MAXAGE: 0, T_SMAXAGE: 0, T_MAXSTALE: 0, T_MINFRESH: 0, PNO$: 0]
[Dec 13 08:58:38.739] DEBUG: (http) FIELD_BLOCK 0x113abc0f8: [T: 5, L:  528, OBJFLAGS: 0]  
[Dec 13 08:58:38.739] DEBUG: (http) [FREETOP: 4, NEXTBLK: 0x0]
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 0 (0x113abc108), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "User-Agent", N_LEN: 10, N_IDX: 64, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5", V_LEN: 78, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 92, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 1 (0x113abc128), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Host", N_LEN: 4, N_IDX: 30, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "localhost", V_LEN: 9, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 2 (0x113abc148), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Accept", N_LEN: 6, N_IDX: 4, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "*/*", V_LEN: 3, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 13, F: 1]
[Dec 13 08:58:38.739] DEBUG: (http) 
[Dec 13 08:58:38.739] DEBUG: (http) 	SLOT # 3 (0x113abc168), LIVE    
[Dec 13 08:58:38.739] DEBUG: (http) [N: "Range", N_LEN: 5, N_IDX: 52, 
[Dec 13 08:58:38.739] DEBUG: (http) V: "bytes=-1", V_LEN: 8, 
[Dec 13 08:58:38.739] DEBUG: (http) NEXTDUP: 0x0, RAW: 1, RAWLEN: 17, F: 1]
```

Still in `HttpTransact::HandleRequest`. We dump the headers with `DUMP_HEADER("http_hdrs", ...)`.

```
[Dec 13 08:58:38.739] DEBUG: (http) 
+++++++++ Incoming Request +++++++++
-- State Machine Id: 6652
GET http://10.10.10.9/DiskStoragePath/Telnet.dmg HTTP/1.1
User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5
Host: localhost
Accept: */*
Range: bytes=-1
```

Still in `HttpTransact::HandleRequest`. At this point, we know the request is acceptable and we are going to process it. We update the `HttpTransact::State` object with the policy from `cache.config`. We call `is_request_cache_lookupable` to determine whether we should try to look up the document in the cache. From here there's a umber of paths we can take, but typically we will either do a DNS lookup or try ti hit the cache immediately.

```
[Dec 13 08:58:38.739] DEBUG: (http_trans) [DecideCacheLookup] Will do cache lookup.
[Dec 13 08:58:38.739] DEBUG: (http_seq) [DecideCacheLookup] Will do cache lookup
```

We get to `HttpTransact::DecideCacheLookup` from the this call chain: `HttpTransact::HandleRequest` → `HttpTransact::StartAccessControl` → `HttpTransact::HandleRequestAuthorized` → HttpTransact::DecideCacheLookup. In this case, we skipped the DNS lookup state and dropped straight into `StartAccessControl`. In `HttpTransact::DecideCacheLookup`, we call `is_request_cache_lookupable` again, and do a bunch more work to figure out whether to attempt the cache lookup. We decide to do it using `TRANSACT_RETURN(CACHE_LOOKUP, ...)`.

```
[Dec 13 08:58:38.739] DEBUG: (http_trans) Next action CACHE_LOOKUP; NULL
[Dec 13 08:58:38.739] DEBUG: (http) [6652] State Transition: HTTP_API_POST_REMAP -&gt; CACHE_LOOKUP
```

We are in `HttpSM::do_cache_lookup_and_read`, having come here from `HttpSM::set_next_state` with `HttpSM::t_state`.next_action equal to `HttpTransact::CACHE_LOOKUP`.

```
[Dec 13 08:58:38.739] DEBUG: (http_seq) [HttpSM::do_cache_lookup_and_read] [6652] Issuing cache lookup for URL http://localhost/DiskStoragePath/Telnet.dmg
```

We call `HttpCacheSM::open_read` to issue the cache read. This function kicks off a cache read via `CacheProcessor::open_read`, and returns an `Action` handle. The continuation is `HttpCacheSM::state_cache_open_read`, which is where the action will pick up.

```
[Dec 13 08:58:38.739] DEBUG: (http_cache) [6652] [&HttpCacheSM::state_cache_open_read, CACHE_EVENT_OPEN_READ_FAILED]
[Dec 13 08:58:38.739] DEBUG: (http) [6652] [HttpSM::main_handler, CACHE_EVENT_OPEN_READ_FAILED]
[Dec 13 08:58:38.739] DEBUG: (http) [6652] [&HttpSM::state_cache_open_read, CACHE_EVENT_OPEN_READ_FAILED]
[Dec 13 08:58:38.739] DEBUG: (http) [6652] cache_open_read - CACHE_EVENT_OPEN_READ_FAILED
[Dec 13 08:58:38.740] DEBUG: (http) [state_cache_open_read] open read failed.
```

The cache read failed. In this case, the most likely error is `ECACHE_NO_DOC`, which means that the document wasn't found in the cache. Receiving the `CACHE_EVENT_OPEN_READ_FAILED` event in `HttpSM::state_cache_open_read` sets the next state to `HttpTransact::HandleCacheOpenRead`.

```
[Dec 13 08:58:38.740] DEBUG: (http) [6652] calling plugin on hook TS_HTTP_CACHE_LOOKUP_COMPLETE_HOOK at hook 0x7fc41a0895e0
[Dec 13 08:58:38.740] DIAG: (lua) LuaDemuxGlobalHook: HTTP_CACHE_LOOKUP_COMPLETE_HOOK lthread=0x7fc419d06570 event=60015 edata=0x1151ba290, ref=1
[Dec 13 08:58:38.740] DIAG: (lua) cache lookup status is TS_CACHE_LOOKUP_MISS
```

We call the `TS_HTTP_CACHE_LOOKUP_COMPLETE_HOOK`  plugin hook, and it turns out that the Lua plugin is listening on that hook and logs some diagnostics.

```
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [&HttpSM::state_api_callback, HTTP_API_CONTINUE]
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [&HttpSM::state_api_callout, HTTP_API_CONTINUE]
[Dec 13 08:58:38.740] DEBUG: (http_trans) [HttpTransact::HandleCacheOpenRead]
[Dec 13 08:58:38.740] DEBUG: (http_trans) CacheOpenRead -- miss
```

We can't be completely sure there was a cache miss, since the cache lookup status is not logged and there are a number of code paths that are all forced to behave as though a cache miss happened, but we are going to handle this as a miss. Because we got a cache miss, we `TRANSACT_RETURN` to the DNS lookup. There is a `HttpTransact::State.force_dns` flag that skips the DNS lookup state. That's the opposite of what I expected. Handling of the `force_dns`flag does not seem to be consistent; there's either some bugs here or some deeper meaning.

```
[Dec 13 08:58:38.740] DEBUG: (http_trans) Next action DNS_LOOKUP; OSDNSLookup
[Dec 13 08:58:38.740] DEBUG: (http) [6652] State Transition: CACHE_LOOKUP -&gt; DNS_LOOKUP
[Dec 13 08:58:38.740] DEBUG: (http_seq) [HttpStateMachineGet::do_hostdb_lookup] Doing DNS Lookup
[Dec 13 08:58:38.740] DEBUG: (http_trans) [HttpTransact::OSDNSLookup] This was attempt 1
```

We are in `HttpTransact::OSDNSLookup`, attempting a DNS lookup for the origin. We got here from `HttpSM::set_next_state` because the next state was `HttpTransact::DNS_LOOKUP`. First, we check whether we are doing SRV record lookups. We aren't, and this is off by default. I guess that no-one uses SRV records for HTTP; I wonder why this was implemented. In `HttpSM::set_next_state`, we pumped the DNS lookup into `HostDBProcessor` and schedule a lookup by name using `HostDBProcessor::getbyname_imm`. I can't really tell from the log, but I think that this hits the DNS cache and executes in line. We call `HttpSM::call_transact_and_set_next_state(NULL)`, which drops us back to the `HttpTransact` state (using `HttpSM::t_state.transact_return_point`), which is `HttpTransact::OSDNSLookup`.

```
[Dec 13 08:58:38.740] DEBUG: (http_seq) [HttpTransact::OSDNSLookup] DNS Lookup successful
[Dec 13 08:58:38.740] DEBUG: (http_trans) [OSDNSLookup] DNS lookup for O.S. successful IP: 10.10.10.9
[Dec 13 08:58:38.740] DEBUG: (http_trans) Next action HttpTransact::HTTP_API_OS_DNS; HandleCacheOpenReadMiss
[Dec 13 08:58:38.740] DEBUG: (http) [6652] State Transition: DNS_LOOKUP -&gt; API_OS_DNS
[Dec 13 08:58:38.740] DEBUG: (http_trans) [HandleCacheOpenReadMiss] --- MISS
```

We are still in` HttpTransact::OSDNSLooku`p, and we did `TRANSACT_RETURN(HttpTransact::HTTP_API_OS_DNS, HandleCacheOpenReadMiss)`. There are 2 conditions that result in this, the most likely of which is `s→cache_lookup_result == CACHE_LOOKUP_MISS || s→cache_info.action == CACHE_DO_NO_ACTION`.

```
[Dec 13 08:58:38.740] DEBUG: (http_seq) [HttpTransact::HandleCacheOpenReadMiss] Miss in cache
```

We are in `HttpTransact::HandleCacheOpenReadMiss` and we can know that the "only-if-cached" cache control was not sent, so we are going to call `how_to_open_connection` to determine our next_action. Before we do that, however, we call` HttpTransact::build_reques`t, which is the source of the next few log lines.

```
[Dec 13 08:58:38.740] DEBUG: (http_trans) client_ip_set = 0
[Dec 13 08:58:38.740] DEBUG: (http_trans) inserted request header 'Client-ip: 127.0.0.1'
[Dec 13 08:58:38.740] DEBUG: (http_trans) [add_client_ip_to_outgoing_request] Appended connecting client's (127.0.0.1) to the X-Forwards header
[Dec 13 08:58:38.740] DEBUG: (http_trans) [build_request] request like cacheable and conditional headers removed
[Dec 13 08:58:38.740] DEBUG: (http_trans) [ink_cluster_time] local: 1355417918, highest_delta: 0, cluster: 1355417918
[Dec 13 08:58:38.740] DEBUG: (http_trans) [build_request] request_sent_time: 1355417918
+++++++++ Proxy's Request +++++++++
-- State Machine Id: 6652
GET /DiskStoragePath/Telnet.dmg HTTP/1.1
User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0 OpenSSL/0.9.8r zlib/1.2.5
Host: localhost
Accept: */*
Range: bytes=-1
Client-ip: 127.0.0.1
X-Forwarded-For: 127.0.0.1
Via: http/1.1 jpeach.example.com[11C91B29] (ApacheTrafficServer/3.3.1-dev [uScMs f p eN:t cCMi p s ])
```

In` HttpTransact::build_reques`t, we dump the headers with `DUMP_HEADER("http_hdrs", ...)`. From the subsequent log lines, we can tell that the result of `how_to_open_connection` was `HttpTransact::CACHE_ISSUE_WRIT`E.

```
[Dec 13 08:58:38.740] DEBUG: (http) [6652] State Transition: API_OS_DNS -&gt; CACHE_ISSUE_WRITE
[Dec 13 08:58:38.740] DEBUG: (http_cache_write) [6652] writing to cache with URL http://localhost/DiskStoragePath/Telnet.dmg
```

We are in` HttpSM::do_cache_prepare_actio`n, from `HttpSM::do_cache_prepare_writ`e. From here, we will call `CacheProcessor::open_writ`e, remembering that the continuation for the cache processor is the `HttpCacheSM` object owned by the current `HttpS`M. `HttpCacheSM` has a back pointer to it's parent `HttpSM` that it uses to switch back into the `HttpSM` state machine. `HttpCacheSM::open_write` sets the next state to `HttpCacheSM::state_cache_open_writ`e.

```
[Dec 13 08:58:38.740] DEBUG: (http_cache) [6652] [&HttpCacheSM::state_cache_open_write, CACHE_EVENT_OPEN_WRITE]
```

HttpCacheSM::state_cache_open_write keeps the CacheVConnection from the cache event, then drops back to the parent HttpSM, using Continuation::handleEvent to pass the event that it received. Because we were paying attention in the prevous step, we noticed that `HttpSM::set_next_state` called `HTTP_SM_SET_DEFAULT_HANDLER(&HttpSM::state_cache_open_write)` prior to entering `HttpSM::do_cache_prepare_write`. So when we pop out of the HttpCacheSM, we end up in `HttpSM::state_cache_open_write`.

```
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [HttpSM::main_handler, CACHE_EVENT_OPEN_WRITE]
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [&HttpSM:state_cache_open_write, CACHE_EVENT_OPEN_WRITE]
[Dec 13 08:58:38.740] DEBUG: (http_trans) Next action next; NULL
[Dec 13 08:58:38.740] DEBUG: (http) [6652] State Transition: CACHE_ISSUE_WRITE -&gt; ORIGIN_SERVER_OPEN
```

Well we ended up getting to `HttpTransact::ORIGIN_SERVER_OPEN`, which most likely was set as a next state from the `HttpTransact::State`, but I can't specifically point out how that happened. We handle `HttpTransact::ORIGIN_SERVER_OPEN` in `HttpSM::set_next_state` as usual and call down into `HttpSM::do_http_server_open`.

```
[Dec 13 08:58:38.740] DEBUG: (http_track) entered inside do_http_server_open ][IPv4]
[Dec 13 08:58:38.740] DEBUG: (http) [6652] open connection to 10.10.10.9: 10.10.10.9:80
```

We are in `HttpSM::do_http_server_open`, and we start trying to get a HTTP session to the origin. There's some checking for origin server congestion in here, but we don't have congestion control enabled, so none of that gets done. The `raw` flag to `HttpSM::do_http_server_open` is used only for connection tunnelling (eg. the CONNECT method), so in this request `raw`is false.

```
[Dec 13 08:58:38.740] DEBUG: (http_seq) [HttpSM::do_http_server_open] Sending request to server
[Dec 13 08:58:38.740] DEBUG: (http) calling netProcessor.connect_re
```

We are now later in `HttpSM::do_http_server_open`, and to get here, we must have failed to obtain an origin server session from the session pool. We call `NetProcessor::connect_re` (the "re" suffix means re-entrant), and start the TCP connection to the origin.

```
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [HttpSM::main_handler, NET_EVENT_OPEN]
[Dec 13 08:58:38.740] DEBUG: (http_track) entered inside state_http_server_open
[Dec 13 08:58:38.740] DEBUG: (http) [6652] [&HttpSM::state_http_server_open, NET_EVENT_OPEN]
```

We are now in `HttpSM::state_http_server_open`, because when we handled `HttpTransact::ORIGIN_SERVER_OPEN`, we set the default even handler to `HttpSM::state_http_server_open` using `HTTP_SM_SET_DEFAULT_HANDLER`. The `NET_EVENT_OPEN` event means that we successfully connected to the origin and we can start sending the request. We call through `HttpSM::handle_http_server_open` and `HttpSM::setup_server_send_request_api` to send the`TS_HTTP_SEND_REQUEST_HDR_HOOK` plugin event, moving to the `HttpTransact::HTTP_API_SEND_REQUEST_HDR` action. In `HttpSM::handle_api_return`, we will call `HttpSM::setup_server_send_request`, which actually marshalls the request and writes it out to the origin. We have a `VConnection` to the origin, and the state we need (marshalll buffers, etc) is wrapped up in a`HttpVCTableEntry` structure. It's typical of plugins to do the same thing when they cann TSHttpConnect. We bounce the `VConnection` events back to the `HttpSM` by setting `HttpVCTableEntry::vc_handler` to `HttpSM::state_send_server_request_header`.

```
[Dec 13 08:58:38.740] DEBUG: (http_ss) [6571] session born, netvc 0x7fc41b813da0
[Dec 13 08:58:38.742] DEBUG: (http) [6652] [HttpSM::main_handler, VC_EVENT_WRITE_COMPLETE]
[Dec 13 08:58:38.742] DEBUG: (http) [6652] [&HttpSM::state_send_server_request_header, VC_EVENT_WRITE_COMPLETE]
```

The write completed and landed us back in `HttpSM::state_send_server_request_header`. We are processing a GET request so we take the path to `HttpSM::setup_server_read_response_header`. We set our`HttpVCTableEntry::vc_handler` to `HttpSM::state_read_server_response_header`and kick off a read operation to start reading the origin response.

```
[Dec 13 08:58:38.744] Server {0x110178000} NOTE: updated diags config
[Dec 13 08:58:38.744] Server {0x110178000} DEBUG: (statsproc) config_update_cont() processed
[Dec 13 08:58:38.897] DEBUG: (http) [6652] [HttpSM::main_handler, VC_EVENT_READ_READY]
[Dec 13 08:58:38.897] DEBUG: (http) [6652] [&HttpSM::state_read_server_response_header, VC_EVENT_READ_READY]
```

We get a VC_EVENT_READ_READY event in HttpSM::state_read_server_response_header and start trying to parse the response data.

```
[Dec 13 08:58:38.897] DEBUG: (socket) Set inactive timeout=30000000000, for NetVC=0x7fc41b813da0
[Dec 13 08:58:38.897] DEBUG: (http_seq) Done parsing server response header
```

We were able to parse the full HTTP response header. This sends us into `HttpSM::do_api_callout` again, and we deliver `TS_HTTP_READ_RESPONSE_HDR_HOOK` to the plugin. We pick up again in `HttpSM::handle_api_return`, calling through into `HttpSM::call_transact_and_set_next_state`which will push us back into HttpTransact.

```
[Dec 13 08:58:38.898] DEBUG: (http_redirect) [HttpTunnel::deallocate_postdata_copy_buffers]
[Dec 13 08:58:38.898] DEBUG: (http_trans) [HttpTransact::HandleResponse]
[Dec 13 08:58:38.898] DEBUG: (http_seq) [HttpTransact::HandleResponse] Response received
[Dec 13 08:58:38.898] DEBUG: (http_trans) [ink_cluster_time] local: 1355417918, highest_delta: 0, cluster: 1355417918
[Dec 13 08:58:38.898] DEBUG: (http_trans) [HandleResponse] response_received_time: 1355417918
+++++++++ Incoming O.S. Response +++++++++
-- State Machine Id: 6652
HTTP/1.1 206 Partial Content
Date: Thu, 13 Dec 2012 16:58:38 GMT
Server: Apache/2.2.22 (Unix) DAV/2 mod_ssl/2.2.22 OpenSSL/0.9.8r
Last-Modified: Tue, 11 Dec 2012 18:24:56 GMT
ETag: "2800b0-5b2873-4d097cc798e00"
Accept-Ranges: bytes
Content-Length: 1
Content-Range: bytes 5974130-5974130/5974131
Content-Type: application/octet-stream
```

We are in `HttpTransact::HandleResponse`, and we dump the received origin response.

```
[Dec 13 08:58:38.898] DEBUG: (http_trans) [is_response_valid] No errors in response
[Dec 13 08:58:38.898] DEBUG: (http_seq) [HttpTransact::HandleResponse] Response valid
[Dec 13 08:58:38.898] DEBUG: (http_hdrs) [initialize_state_variables_from_response]Server is keep-alive.
```

After doing some syntactic checking and deciding that we have a valid response, we call into `HttpTransact::initialize_state_variables_from_response`and parse a ton of state from the response. This is where we figure out the transfer encoding and deal with it if we understand it.

```
[Dec 13 08:58:38.898] DEBUG: (http_trans) [handle_response_from_server] (hrfs)
[Dec 13 08:58:38.898] DEBUG: (http_trans) [hrfs] connection alive
[Dec 13 08:58:38.898] DEBUG: (http_trans) [handle_forward_server_connection_open] (hfsco)
[Dec 13 08:58:38.898] DEBUG: (http_seq) [HttpTransact::handle_server_connection_open] 
[Dec 13 08:58:38.898] DEBUG: (http) server info = 10.10.10.9:80
[Dec 13 08:58:38.898] DEBUG: (http_trans) [hfsco] cache action: CACHE_DO_WRITE
[Dec 13 08:58:38.898] DEBUG: (http_trans) [handle_cache_operation_on_forward_server_response] (hcoofsr)
[Dec 13 08:58:38.898] DEBUG: (http_seq) [handle_cache_operation_on_forward_server_response]
[Dec 13 08:58:38.898] DEBUG: (http_trans) [is_response_cacheable] client permits storing
[Dec 13 08:58:38.898] DEBUG: (http_trans) [is_response_cacheable] Partial content response - don't cache
[Dec 13 08:58:38.898] DEBUG: (http_trans) [hcoofsr] response code: 206
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] age_value:              0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] date_value:             1355417918
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] response_time:          1355417918
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] now:                    1355417918
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] now (fixed):            1355417918
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] apparent_age:           0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] corrected_received_age: 0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] response_delay:         0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] corrected_initial_age:  0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] resident_time:          0
[Dec 13 08:58:38.898] DEBUG: (http_age) [calculate_document_age] current_age:            0
[Dec 13 08:58:38.898] DEBUG: (http_trans) [handle_content_length_header] RESPONSE cont len in hdr is 1
[Dec 13 08:58:38.898] DEBUG: (http_trans) [WUTS code generation] Hit/Miss: 49, Log: 51, Hier: 50, Status: 206
[Dec 13 08:58:38.898] DEBUG: (http_trans) Adding Server: ATS/3.3.1-dev
```
```
+++++++++ Base Header for Building Response +++++++++
-- State Machine Id: 6652
HTTP/1.1 206 Partial Content
Date: Thu, 13 Dec 2012 16:58:38 GMT
Server: Apache/2.2.22 (Unix) DAV/2 mod_ssl/2.2.22 OpenSSL/0.9.8r
Last-Modified: Tue, 11 Dec 2012 18:24:56 GMT
ETag: "2800b0-5b2873-4d097cc798e00"
Accept-Ranges: bytes
Content-Length: 1
Content-Range: bytes 5974130-5974130/5974131
Content-Type: application/octet-stream
```
```
+++++++++ Proxy's Response 2 +++++++++
-- State Machine Id: 6652
HTTP/1.1 206 Partial Content
Date: Thu, 13 Dec 2012 16:58:38 GMT
Server: ATS/3.3.1-dev
Last-Modified: Tue, 11 Dec 2012 18:24:56 GMT
ETag: "2800b0-5b2873-4d097cc798e00"
Accept-Ranges: bytes
Content-Length: 1
Content-Range: bytes 5974130-5974130/5974131
Content-Type: application/octet-stream
Age: 0
Connection: keep-alive
Via: http/1.1 jpeach.example.com (ApacheTrafficServer/3.3.1-dev [uScMsSf pSeN:t cCMi p sS])
```
```
[Dec 13 08:58:38.898] DEBUG: (http) [6652] State Transition: ORIGIN_SERVER_OPEN -&gt; SERVER_READ
[Dec 13 08:58:38.898] DEBUG: (http_redirect) [HttpSM::do_redirect]
[Dec 13 08:58:38.898] DEBUG: (http_redirect) [HttpTunnel::deallocate_postdata_copy_buffers]
[Dec 13 08:58:38.898] DEBUG: (socket) Set inactive timeout=30000000000, for NetVC=0x7fc41b814820
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) [6652] adding producer 'http server'
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) [6652] adding consumer 'user agent'
[Dec 13 08:58:38.898] DEBUG: (http) [6652] perform_cache_write_action CACHE_DO_NO_ACTION
[Dec 13 08:58:38.898] DEBUG: (cache_free) free 0x7fc41a29ce40
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) tunnel_run started, p_arg is NULL
[Dec 13 08:58:38.898] DEBUG: (http_cs) tcp_init_cwnd_set 0
[Dec 13 08:58:38.898] DEBUG: (http_cs) desired TCP congestion window is 0
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) [6652] [tunnel_run] producer already done
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) [6652] producer_handler [http server HTTP_TUNNEL_EVENT_PRECOMPLETE]
[Dec 13 08:58:38.898] DEBUG: (http_redirect) [HttpTunnel::producer_handler] enable_redirection: [0 0 0] event: 2302
[Dec 13 08:58:38.898] DEBUG: (http) [6652] [&HttpSM::tunnel_handler_server, HTTP_TUNNEL_EVENT_PRECOMPLETE]
[Dec 13 08:58:38.898] DEBUG: (http_cs) [6577] attaching server session [6571] as slave
[Dec 13 08:58:38.898] DEBUG: (socket) Set inactive timeout=120000000000, for NetVC=0x7fc41b813da0
[Dec 13 08:58:38.898] DEBUG: (http_tunnel) [6652] consumer_handler [user agent VC_EVENT_WRITE_COMPLETE]
[Dec 13 08:58:38.898] DEBUG: (http) [6652] [&HttpSM::tunnel_handler_ua, VC_EVENT_WRITE_COMPLETE]
[Dec 13 08:58:38.898] DEBUG: (http_cs) [6577] session released by sm [6652]
[Dec 13 08:58:38.898] DEBUG: (http_cs) [6577] initiating io for next header
[Dec 13 08:58:38.898] DEBUG: (socket) Set inactive timeout=115000000000, for NetVC=0x7fc41b814820
[Dec 13 08:58:38.898] DEBUG: (http) [6652] [HttpSM::main_handler, HTTP_TUNNEL_EVENT_DONE]
[Dec 13 08:58:38.898] DEBUG: (http) [6652] [&HttpSM::tunnel_handler, HTTP_TUNNEL_EVENT_DONE]
[Dec 13 08:58:38.898] DEBUG: (http_redirect) [HttpTunnel::deallocate_postdata_copy_buffers]
[Dec 13 08:58:38.898] DEBUG: (http_seq) [HttpStateMachineGet::update_stats] Logging transaction
[Dec 13 08:58:38.898] DEBUG: (http) [6652] dellocating sm
[Dec 13 08:58:38.899] DEBUG: (http_cs) [6577] [&HttpClientSession::state_keep_alive, VC_EVENT_EOS]
[Dec 13 08:58:38.899] DEBUG: (socket) Set inactive timeout=120000000000, for NetVC=0x7fc41b813da0
[Dec 13 08:58:38.899] DEBUG: (socket) Set active timeout=0, NetVC=0x7fc41b813da0
[Dec 13 08:58:38.899] DEBUG: (http_ss) [6571] [release session] session placed into shared pool
[Dec 13 08:58:38.899] DEBUG: (http_cs) [6577] session closed
[Dec 13 08:58:38.899] DEBUG: (http_cs) [6577] session destroy
```
