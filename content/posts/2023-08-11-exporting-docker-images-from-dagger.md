---
title: "Exporting Docker images from Dagger"
date: 2023-08-11T08:50:45+10:00
tags: [docker,dagger,containers]
---

I started using [Dagger](https://dagger.io/) this week, and if you
have any sort of build and test system based on shell scripts and
Dockerfiles, Dagger will be a big improvement.

This post documents how to export a container image that you build
in Dagger to your local Docker instance. This process is described
in the
[Dagger documentation](content/posts/2023-08-11-exporting-docker-images-from-dagger.md)
but I needed to go one step further and tag the image that I exported.

The first step is to actually export the container image from Dagger. This
will write an OCI tarball to the local filesystem.

```Go
var container *dagger.Container

ok, err := container.Export("path/to/container-image.tgz")
if err != nil {
        log.Fatal(err.Error())
}
if !ok {
        log.Fatal("container export failed")
}
```

Now you have tarball containing an OCI, image, you have to import it into
Docker using "docker load", *not* "docker import"
(see [#3710](https://github.com/dagger/dagger/issues/3710) for example).
However, "docker import" doesn't let you tag the image at import time, which
is awkward because no-one wants to refer to images just by their ID.

```Go
buildImagePath := "path/to/container-image.tgz"
cmdImport := exec.Command("docker", "load", "--quiet", "--input", buildImagePath)
out, err := cmdImport.CombinedOutput()
if err != nil {
        log.Fatalf("failed to export builder image: %s", err.Error())
}
```

Note that in the command above, I used the `--quiet` flag. When this is set,
Docker will write a single lin eof output containing the hash of the image that
it imported. We can scrape the container ID from this output and use it to
tag the image that we just created.

```Go
r := regexp.MustCompile("Loaded image ID: ([:a-z0-9]+)")
matches := r.FindStringSubmatch(string(out))
if len(matches) == 0 {
    log.Fatalf("failed to match image ID from Docker output %q", string(out))
}

repoName := "container-image:latest"
cmdTag := exec.Command("docker", "tag", matches[1], repoName)
if out, err := cmdTag.CombinedOutput(); err != nil {
    log.Fatalf("failed to tag builder image: %s", string(out))
}

log.Printf("tagged build container image %q as %q", matches[1], repoName)

```

