---
title: "Building Dreamcast programs With GCC spec files"
date: 2023-05-25T13:53:13+10:00
subtitle: ""
tags: [gcc,dreamcast,kallistios]
---

In the KallistiOS ecosystem, there are basically two ways to build
Dreamcast programs - use the CMake toolchain support, or use the compiler
wrapper scripts from the KallistiOS source tree. I was interested in the
latter, but they end up depending on a large number of environment variables,
which are traditionally sourced through `$KOS_BASE/environ.sh`. Although it's
not really a big issue to have all those environment variables, I wondered
whether there was a cleaner approach.

GCC has a [spec file](https://gcc.gnu.org/onlinedocs/gcc/Spec-Files.html)
feature, which is a custom DSL that describes how the compiler driver
should process flags for the various programs in the compilation
toolchain. Since the KallistiOS compiler wrappers are basically injecting
compiler flags, spec files seemed like a promising approach.

It turns out that this approach works pretty nicely, and we only need to depend
on the single `$KOS_BASE` environment variable being set. I'm pretty naive
about GCC internals, but the spec file below worked for me.

```
*kos_header_paths:
-isystem %:getenv(KOS_BASE /include) -isystem %:getenv(KOS_BASE /kernel/arch/dreamcast/include) -isystem %:getenv(KOS_BASE /addons/include)

*kos_library_paths:
-L%:getenv(KOS_BASE /lib/dreamcast) -L%:getenv(KOS_BASE /addons/lib/dreamcast)

*kos_linker_script:
-T%:getenv(KOS_BASE /utils/ldscripts/shlelf.xc)

*kos_libs:
--start-group -lkallisti -lc %G --end-group

*kos_defines:
-D_arch_dreamcast

*kos_cc_options:
-ffunction-sections -fdata-sections -fno-builtin

*cc1_options:
+ %(kos_header_paths) %(kos_defines) %(kos_cc_options)

*cc1plus_options:
+ %(kos_header_paths) %(kos_defines) %(kos_cc_options)

*cpp_options:
+ %(kos_header_paths) %(kos_defines)

*subtarget_link_spec:
%(kos_linker_script) -Ttext=0x8c010000 --gc-sections

*link_gcc_c_sequence:
%(kos_library_paths) %(kos_libs)
```

I think that this makes it a lot simpler to build Dreamcast programs. In the
trivial case, it can be as easy as this:

```bash
$ /usr/local/Cellar/dc-toolchain-testing/2023.05.19/sh-elf/bin/sh-elf-g++ -specs=kos-dreamcast.specs -o hello.elf hello.cpp
$ file hello.elf
hello.elf: ELF 32-bit LSB executable, Renesas SH, version 1 (SYSV), statically linked, with debug_info, not stripped
```

The final benefit is that this is easy to use in any old build system. You
just have to add the `-specs` flag to your compiler flags, in what ever
way the build system supports.

