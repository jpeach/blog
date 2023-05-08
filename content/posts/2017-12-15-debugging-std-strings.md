---
title: "Debugging libstdc++ strings"
date: 2017-12-15T19:27:21+10:00
tags: [linux,debugging,c++]
featured_image: ""
description: ""
---

Writing this down quickly before I forget.

When debugging a `std::string` from GNU libstdc++, the debugger typically
won't show you the actual representation.

First, you need to turn off the [pretty
printer](https://sourceware.org/gdb/onlinedocs/gdb/Pretty_002dPrinter-Commands.html)
(assuming that it worked in the first place):

```
(gdb) p reregisterSlaveMessage.resource_version_uuid_.ptr_
$13 = (std::string *) 0x7f4d98d5e970
(gdb) p *reregisterSlaveMessage.resource_version_uuid_.ptr_
$14 = "\022\020K|\n\225\064\246CE\222\350\275\315t", <incomplete sequence>
(gdb) disable pretty-printer
2 printers disabled
0 of 2 printers enabled
(gdb) p *reregisterSlaveMessage.resource_version_uuid_.ptr_
$15 = {
  static npos = <optimized out>,
  _M_dataplus = {
    <:allocator>> = {
      <:new_allocator>> = {<no data fields>}, <no data fields>},
    members of std::basic_string<char std::char_traits>, std::allocator<char> >::_Alloc_hider:
    _M_p = 0x7f4d98d67068 "\022\020K|\n\225\064\246CE\222\350\275\315t", <incomplete sequence>
  }
}
```

Next, you need to know that the internal structure of `std::string` is
prepended to the actual string data, so you need to cast and subtract
from the data pointer to find the length and refcount. In the example
below, I show the `std::string::_Rep` structure that is laid out in
memory immediately before the `_M_p` string data pointer:

```
(gdb) p (((std::string::_Rep *)0x7f4d98d67068) - 1)[0]
$16 = {
  <:basic_string std::char_traits>, std::allocator<char> >::_Rep_base> = {
    _M_length = 18,
    _M_capacity = 18,
    _M_refcount = -1
  },
  members of std::basic_string<char std::char_traits>, std::allocator<char> >::_Rep:
  static _S_max_size = <optimized out>,
  static _S_terminal = <optimized out>,
  static _S_empty_rep_storage = <optimized out>
}
``` 
