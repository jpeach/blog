---
title: "Being too clever merging protobufs"
date: 2019-10-23T00:00:00+10:00
tags: [go,protobuf]
featured_image: ""
description: ""
---

This is probably something lots of other people have tried and burnt
themselves with, but anyway, this time it’s my turn.

The goal is, given an arbitrary protobuf, can we write an API that
applies default values to it? Normally we would create a prototype
protobuf object with the defaults and merge our current object into it,
updating the prototype object with current values. However, in Go, this
would erase the type information and mean we would have to do some ugly
casting (it’s easy to avoid this in C++ by using templates).

So me, being clever, comes up with this construct (in Go):

```Go
message := proto.Message{} // The message to update
prototype := SomeStruct{} // The defaults to apply

// Merge the message into the prototype, which updates any
// default values with values from the message, and retains
// any that the message doesn't specify.
proto.Merge(prototype, message)

// Now here's the trick. We update the message with the updated
// prototype. The prototype is (potentially) a superset of the
// message since it has filled in missing default values, but
// stil has the same values it copied from the message.
proto.Merge(message, prototype)
```

The reason this doesn’t work is that merging repeated fields
concatenates them. So if the original message had any repeated fields,
the double merge just doubled them.

And that’s how I spent the afternoon debugging.
