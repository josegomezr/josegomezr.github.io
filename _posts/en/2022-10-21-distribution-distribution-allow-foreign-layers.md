---
title: "TIL: distribution/distribution doesn't allow foreign layers by default"
layout: post
tags: [blog]
categories: [blog, til]
lang: 
---

TIL: distribution/distribution doesn't allow foreign layers by default. You must provide an allow/deny list in the configuration.

```yaml
validation:
  manifests:
    urls:
      allow:
        -  ^https://your.fancy.domain.internal/
```

<!--more-->

Say you have a container image which has foreign layers in it (very very frequent in windows). Say this image is
idk: `rancher/kubelet-pause@sha256:6e1d6e94d15c837e1b0361b435cff67334ec71529a9ab6f1ba191a45e12e63fb` and you want
to push it to your own container registry for containery purposes.

WELP, you'll be cursed with a nice error message like this:

```
failed to put manifest my-registry.internal/rancher/kubelet-pause@sha256:6e1d6e94d15c837e1b0361b435cff67334ec71529a9ab6f1ba191a45e12e63fb: request failed: unexpected http status code: Internal Server Error [http 500]: {"errors":[{"code":"UNKNOWN","message":"unknown error"},{"code":"UNKNOWN","message":"unknown error","detail":{}},{"code":"MANIFEST_BLOB_UNKNOWN","message":"blob unknown to registry","detail":"sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9"}]}
```

and in the backend you get:

```
data.time: 2022-10-20T14:36:41.421276552Z
data.err.code: unknown
data.err.message: invalid URL on layer
data.err.detail: rancher/fleet-agent
```

but wait... invalid? Let's checkout this image:

```
docker pull rancher/kubelet-pause@sha256:6e1d6e94d15c837e1b0361b435cff67334ec71529a9ab6f1ba191a45e12e63fb
docker.io/rancher/kubelet-pause@sha256:6e1d6e94d15c837e1b0361b435cff67334ec71529a9ab6f1ba191a45e12e63fb: Pulling from rancher/kubelet-pause
62239e9aa1a3: Pulling fs layer 
8cfe6e1c44d6: Pulling fs layer 
9a57ef28b615: Pulling fs layer 
8a046a3e9f52: Waiting 
5ee4579af890: Waiting 
52d78f3e36b1: Waiting 
image operating system "windows" cannot be used on this platform
```

perfect 🤦‍♂ ok, let's inspect the manifests then. (used [`reg`](https://github.com/genuinetools/reg) from https://github.com/genuinetools/reg)

```
reg manifest rancher/kubelet-pause@sha256:6e1d6e94d15c837e1b0361b435cff67334ec71529a9ab6f1ba191a45e12e63fb
INFO[0000] domain: docker.io                            
INFO[0000] server address: https://registry-1.docker.io 
```
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 2869,
    "digest": "sha256:895a4b97815fd1c70eaac96dd7fb49ce6b4d402b72ccced4b49ccd1e301f8b24"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.foreign.diff.tar.gzip",
      "size": 101340070,
      "digest": "sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9",
      "urls": [
        "https://mcr.microsoft.com/v2/windows/nanoserver/blobs/sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9"
      ]
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 831,
      "digest": "sha256:8cfe6e1c44d6f26710204fa8f3fb990c36545485dc649c09e3fdf2cf6f090de2"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 855,
      "digest": "sha256:9a57ef28b615ffcb64f9dbbfb680c75509653f1244b8216c60646e6812f871e7"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 833,
      "digest": "sha256:8a046a3e9f5285da8c5c5a8cb7fedfcd077b7fb66023877d569269af8d40e346"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 831,
      "digest": "sha256:5ee4579af890bd4fd548e36eb1c966a190f5fe7ba918dac35257856c119e9f48"
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 855,
      "digest": "sha256:52d78f3e36b1ca599d15241c3509bb3ebb857f79f2c07d830edd6f743a538545"
    }
  ]
}
```

Aha! First layer there (`62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9`) corresponds with the error.

```json
{"ommitted": "for brevity"},
    {
      "mediaType": "application/vnd.docker.image.rootfs.foreign.diff.tar.gzip",
      "size": 101340070,
      "digest": "sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9",
      "urls": [
        "https://mcr.microsoft.com/v2/windows/nanoserver/blobs/sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9"
      ]
    },
{"ommitted": "for brevity"}
```
I see a very valid URL there, and it's also reachable:

```
curl -I "https://mcr.microsoft.com/v2/windows/nanoserver/blobs/sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9"
HTTP/2 200 
content-length: 101340070
content-type: application/octet-stream
access-control-expose-headers: Docker-Content-Digest
access-control-expose-headers: WWW-Authenticate
access-control-expose-headers: Link
access-control-expose-headers: X-Ms-Correlation-Request-Id
docker-content-digest: sha256:62239e9aa1a352a20b0d4969c2b508b8a18d10e799d4db72e6f24ccbb2c724d9
docker-distribution-api-version: registry/2.0
# ...
```

Let's look into `distribution/distribution` source code for `invalid URL on layer`, maybe that can shed a light?

After some digging, [here in registry/storage/schema2manifesthandler.go#L108](https://github.com/distribution/distribution/blob/9329f6a62b67d5e06d50dc93997c7705a075fcd9/registry/storage/schema2manifesthandler.go#L108) 

there's a loooong if clause that reads:

```golang
// ...
var pu *url.URL
pu, err = url.Parse(u)
if err != nil || (pu.Scheme != "http" && pu.Scheme != "https") || pu.Fragment != "" || (allow != nil && !allow.MatchString(u)) || (deny != nil && deny.MatchString(u)) {
    err = errInvalidURL
    break
}
// ...
```

Let's break up styling to read that line better:

```golang
if 
err != nil // url couldn't be parsed
|| ( pu.Scheme != "http" && pu.Scheme != "https") // URL schema is not http/https
|| pu.Fragment != "" // there's a fragment in the url z.B: http://domain.tld/path?query=string#fragment
|| (allow != nil && !allow.MatchString(u)) // there's an allow list and it doesn't match
|| (deny != nil && deny.MatchString(u)) // there's a deny list and it matches
```

If any of those conditions matches, I get the error reported above. And analyzing it:

1. the URL is valid ✅
2. it's https ✅
3. has no fragment ✅
4. theres not an allow list¹ ✅
5. there's not a deny list¹ ✅


¹ that I'm aware of! As usual the devil is in the details. After some more digging the root cause appeared!

In [registry/handlers/app.go#L232](https://github.com/distribution/distribution/blob/78b9c98c5c31c30d74f9acb7d96f98552f2cf78f/registry/handlers/app.go#L232)

```golang
// ...
if len(config.Validation.Manifests.URLs.Allow) == 0 && len(config.Validation.Manifests.URLs.Deny) == 0 {
  // If Allow and Deny are empty, allow nothing.
  options = append(options, storage.ManifestURLsAllowRegexp(regexp.MustCompile("^$")))
}
// ...
```

There it is! A default allow nothing blocks all foreign layers from the container distribution service by default. The fix is simple, just configure an allow/deny list and you should be good to go. For our intent something along the lines of:

```yaml
validation:
  manifests:
    urls:
      allow:
        -  ^https://mcr.microsoft.com/
```

Should suffice, but YMMV!

P.S: Don't trust things here to be in production for christ sake...
