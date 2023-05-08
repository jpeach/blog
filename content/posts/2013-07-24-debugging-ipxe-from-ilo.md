---
title: "Debugging iPXE from the iLO console"
date: 2013-07-24T15:44:00+10:00
tags: [linux,ipxe,ilo,console]
featured_image: ""
description: ""
---

So I've been trying to get [iPXE chainloading](http://ipxe.org/howto/chainloading)
to work and I've
been using the iLO virtual serial console over SSH to verify and
debug. iPXE has a DCHP debug build option which you can enable by doing
`make bin/undionly.kpxe DEBUG=dhcp`. However, when you do this, you will
find that each line of output on the iLO virtual serial console output
overwrites the previous line, creating a big illegible mess. Fortunately,
you can build iPXE with only serial output support, so that you
can actually read the debug messages on the iLO virtual serial console.

The [iPXE build options](http://ipxe.org/buildcfg) include various settings to control where output is sent. What you want to to is configure iPXE so that it sends output to the serial console and not to the PC BIOS console. We can do this by editing two local configuration header files.

First, edit `src/config/console.h` to turn off the PC BIOS output and to
the serial console. We keep the PC BIOS enabled for the user interface,
which lets the [iPXE command line](http://ipxe.org/cmdline) work:

```
[root@localhost ipxe.git]# cat src/config/local/console.h
#undef CONSOLE_PCBIOS
#undef CONSOLE_SERIAL
#undef LOG_LEVEL

#define CONSOLE_PCBIOS CONSOLE_USAGE_TUI
#define CONSOLE_SERIAL (CONSOLE_USAGE_ALL & ~CONSOLE_USAGE_TUI)
#define LOG_LEVEL LOG_ALL
```

Next, edit `src/config/local/serial.h` to specify the correct serial port. It's highly probable that you want COM2:

```
[root@localhost ipxe.git]# cat src/config/local/serial.h 
#undef COMCONSOLE

#define COMCONSOLE COM2
```

Finally, if you have a network that's anything like mine, you will want to increase the DHCPOFFER timout by applying this change:

```
[root@localhost ipxe.git]# git diff
diff --git a/src/include/ipxe/dhcp.h b/src/include/ipxe/dhcp.h
index 6c02846..ca35206 100644
--- a/src/include/ipxe/dhcp.h
+++ b/src/include/ipxe/dhcp.h
@@ -645,7 +645,7 @@ struct dhcphdr {

 /** Timeouts for sending DHCP packets */
 #define DHCP_MIN_TIMEOUT ( 1 * TICKS_PER_SEC )
-#define DHCP_MAX_TIMEOUT ( 10 * TICKS_PER_SEC )
+#define DHCP_MAX_TIMEOUT ( 64 * TICKS_PER_SEC )

 /** Maximum time that we will wait for ProxyDHCP responses */
 #define PROXYDHCP_MAX_TIMEOUT ( 2 * TICKS_PER_SEC )
```
