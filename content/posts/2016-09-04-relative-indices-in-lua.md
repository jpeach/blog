---
title: "Dealing with relative indices in Lua APIs"
date: 2016-09-04T13:34:14+10:00
tags: [lua]
featured_image: ""
description: ""
---

When you use the Lua C API to implement custom
Lua bindings, you inevitably end up with internal
helper functions that accept a Lua stack index.
[Lua stack indicies](https://www.lua.org/manual/5.1/manual.html#3.1)
can be positive, which indicates an index from the bottom of the stack or
negative, which is an index from the top of the stack. It is extremely
common to pass `-1` to functions to indicate they should operate on
the value at the top of the stack.

Now, if the function that accepts the index also needs to manipulate the
stack, we have a problem. Any manipulation of the stack will invalidate
negative indices that we have received as arguments. Previously, I have
dealt with this by converting the negative index to a positive absolute
index. The code below is cribbed from `abs_index` in the Lua source:

```C
int
lua_absolute_index(lua_State *L, int relative)
{
  return (relative > 0 || relative <= LUA_REGISTRYINDEX)
    ? relative
    : lua_gettop(L) + relative + 1;
}
```

Recently, however, I realized that the problem is not that you have a
relative index, but that you don’t know which relative index it is. If
you `lua_pushvalue` the index you receive as an argument, then you now have
the value at index `-1` and you can deal with it as you would normally
deal with any stack values. This technique seems a little cleaner than
converting to absolute indicies.
