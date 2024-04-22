---
title: "~~docker~~ OCI distribution storage 101"
layout: post
tags: [blog, til, docker, storage, containers]
categories: [blog]
lang: 
---

Or how to understand this sparse in-disk pointer table?

![OCI Storage Layout Overview]({{ '/images/oci-layout-overview.jpg' | relative_url }})
{: class="text-center" }

<!-- more -->

# It all starts with a blob...

![Blob Storage Layout Overview]({{ '/images/oci-blob-layout.jpg' | relative_url }})
{: class="text-center" }

I've been digging deeply into this as pulling things from OCI registries is becoming more common in may day-to-day.

This is truly a master piece of design for concurrency & throughput that was designed to operate on disk... not on our _beloved_ (/s) Cloud Environments.

As the title says, it all starts on BLOBs [Binary Large Objects], it's the bedrock of an OCI registry. Blobs are stored in the root `blobs/` folder following the pattern:

```
blobs/<digest>/<short-hex>/<hex>
```

* `digest`: The digest algorithm used. It just so happens that `sha256` is the default now, but the layout support arbitrary digests & even settings for it, like: `sha256+b64` for "sha256 base64 encoded" in case that's useful (?).
* `short-hex`: The first two hex chars of the digest. _I speculate, but I think it migh related to folder limit on a file system?_
* `hex`: The actual `<digest>` result in hex (at least for `sha256`).

The beauty of the design is that every piece of data uploaded to the registry must be sent with a digest (that is in current times: `sha256sum` of the content). If the digest sent matches something in the storage then we're receiving _the same content_ and don't need to store it again.

However this alone is a bit useless on it's own, there's no way of knowing what BLOB belongs to what, or how may they be related. Enter... the _repository structure_

# Tag'em all! I mean, Repo'em all?

The juicy part is the `repositories` folder, where three big things happen:

1. The definition of a "repository".
2. The definition of tags of a repository.
3. The definition of "known blobs", and these are scoped per-repository.

![Repository Storage Layout Overview]({{ '/images/oci-manifest-layout.jpg' | relative_url }})
{: class="text-center" }

## Definition of "repository"

A "repository" is nothing but a folder that matches the following structure:

```
repositories/<path>/_manifests/
```

* `path` can be arbitrarily long (I couldn't spot any limitation on this so far).

The primary requirement is that after any number of directories under `repositories/`, there's one `_manifests/`.

For example: A repository `my/repo` would look like this in the FS:

```
repositories/my/repo/_manifests/
```

## Definition of "tag"

A "repository" is nothing but a folder that matches the following structure:

```
repositories/<path>/_manifests/tags/<tag>/current/link
```

* `path` can be arbitrarily long (as repository).
* `tag` must follow  roughly this grammar: `\w[\w_]{1,127}`.

In contrast to repositories, tags can't be nested.

The content of the `link` file is the digest of the blob it represents.

For example: `my/repo:latest` would look like this in the FS:

```
repositories/my/repo/_manifests/tags/latest/current/link
sha256:cafe
```

And that would mean a "link" to:

```
blobs/ca/cafe/data
```


In contrast to BLOBs, tags must be compliant with a certain amount of mime types. Including (and probably missing some others):

- ManifestList // OCI Index
  + `application/vnd.docker.distribution.manifest.list.v2+json`
  + `application/vnd.oci.image.index.v1+json`
- Image Manifest
  + `application/vnd.docker.distribution.manifest.v2+json`
  + `application/vnd.oci.image.manifest.v1+json`

and it's indicated inside of the JSON blob in the root `mediaType` key.

## Definition of "known blobs"

The service _also_ keeps track of what blobs belongs to which repositories via the following structure:

```
repositories/<path>/_layers/<digest>/<hex>/link
```

* `path` can be arbitrarily long (as repository).
* `digest`: same rules as with BLOBs above.

This file structure is used for maintenance tasks like garbage collection (a mark & sweep operation for removing unreferenced blobs. A task I've never been able to complete on a 5TB registry backed by S3).


All of it together looks like:

![OCI Storage Layout Overview]({{ '/images/oci-storage-layout.jpg' | relative_url }})
{: class="text-center" }
