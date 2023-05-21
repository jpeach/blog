---
title: "Homebrew tricks for the Dreamcast toolchain"
date: 2023-05-21T11:08:46+10:00
subtitle: ""
tags: [homebrew,dreamcast,kallistios]
---

I've spent some time working on a
[Homebrew tap](https://github.com/jpeach/homebrew-dreamcast)
for Dreamcast development tooling, and wanted to write a little about the
tricks I used in creating formula to install the Dreamcast compilation
toolchain.

First, the Dreamcast toolchain is actually build and installed by the
[dc-chain][1]
package from the KallistiOS repository.
[dc-chain][1] has 3 phases - download, unpack and build. The download phase is
done by the `download.sh` script, which downloads the source archives of the
toolchain components (gcc, binutils, newlib, gdb) for SH4 and ARM
architectures. The `unpack.sh` script then extracts the source archives, fusr
deleting any previous extractions. It also has the side-effect of downloading
GCC prerequisites that are specified in the GCC sources. Finally, the build
phase is implemented in make, and builds and installs the toolchain components.
All the settings for this, including installation path and the versions of the
components are given in a file named `config.mk`.

For the Homebrew formula, we have 3 loose requirements:
1. a separate formula for the upstream toolchain legacy, stable and testing
   configurations
3. to automatically detect the source archives to download
1. for Homebrew to do the downloads, so we get caching and checksums for free

Our first requirement is easy. We just generate the three forumlas. My first
pass just used `envsubst`, but I switched to a full-blown
[shell script](https://github.com/jpeach/homebrew-dreamcast/blob/main/bin/generate-toolchain-formula.sh)
when it got a bit complicated.

For the second one, we take advantage of the fact that [dc-chain][1] knows the
source archive URLs, and can tell us. We pull the KallistiOS source (as a
tarball so that we don't need to fetch the full history), and twiddle the shell
scripts to output
[Homebrew resources](https://docs.brew.sh/Formula-Cookbook#specifying-gems-python-modules-go-projects-etc-as-dependencies)
for the URLs that the configuration would download.
Since the same sources can be shared between SH4 and ARM, we deduplicate on the base
filename so that we only download archives once.

Note that, [dc-chain][1] will still download some extra sources; this just
captures the majority.

```bash
  # Generate a resources fragment for this variant.
  (
    cd "${WORKDIR}/utils/dc-chain"

    ln -sf config.mk.${variant}.sample config.mk

    # shellcheck disable=SC1091
    source scripts/common.sh > /dev/null

    # Hash of name to URL, for deduplication.
    declare -A resources

    for prefix in "SH_BINUTILS" "SH_GCC" "NEWLIB" "ARM_BINUTILS" "ARM_GCC" "GDB" ; do
      url="${prefix}_TARBALL_URL"
      filename=$(basename "${!url}")
      resources["${filename}"]="${!url}"
    done

    for r in "${!resources[@]}" ; do
      cat <<EOF
  resource "${r}" do
    url "${resources[$r]}"
  end
EOF
    done
  )
```

The last piece took a little experimentation. The normal Homebrew approack is
to use the `.stage` method on the forumula resources. This unpacks the resource
into a private directory and lets you operate on it from there. However, we
really want the `unpack.sh` script to do the extraction, because it has the
side-effect of also downloading GCC prerequisites. The solution is to fish
around in the Homebrew cache directly:

```ruby
      resources.each do |r|
        cp r.cached_download, buildpath/"utils/dc-chain/#{r.name}"
      end
```

We know that the name of the resource is the archive filenames, so we can
directly copy from the cache to the toolchain build directory, and let
`unpack.sh` process it.

The only thing missing is the SHA256 checksums for the resources. I didn't add
that because I was too lazy to wait for all the source archives to download
each time I want to regenerate the formulas.

[1]: https://github.com/KallistiOS/KallistiOS/tree/master/utils/dc-chain
