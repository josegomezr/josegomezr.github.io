

---
title: Is this how you use "strong parameters"?
layout: post
tags: [blog, rails, ruby]
categories: [blog, programming]
lang: en
---

_old man yells at cloud.gif_

Is this how you use `ActionController::Parameters` ?

If this reaches anybody that can respond this, please reach me on ~~Twitter~~ X @ErCuatrista 

---

I wanna validate in pure rails the following request body from a docker distribution notification:

```json
{
  "events": [{
    "id": "551a8b80-461c-49b3-a8b7-116f4c9a29f7",
    "timestamp": "2024-03-05T09:26:03.989254399Z",
    "action": "push",
    "target": {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:da883dc525d7c43fb5d51ce10229f48bb286c6f0a5d733aeaf46b4774a77a7da",
      "size": 928,
      "length": 928,
      "repository": "busybox",
      "url": "https://my-custom-registry.internal/v2/busybox/manifests/sha256:da883dc525d7c43fb5d51ce10229f48bb286c6f0a5d733aeaf46b4774a77a7da",
      "tag": "v1.4",
      "references": [
        {
          "mediaType": "application/vnd.oci.image.config.v1+json",
          "digest": "sha256:0a67789e210f578276f4300c1375b672edf8c761a62e4172797d99a7eb785e90",
          "size": 4818
        },
        {
          "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
          "digest": "sha256:c7d92aa556ade85fe6ceb84d125039873c1987257a251ee3105a18c35077e3ed",
          "size": 10465060
        },
        {
          "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
          "digest": "sha256:e201b69a721b500c261846196f7f3fba33afdf5ed2c52b4d885717c78d7cdae8",
          "size": 6436852
        },
        {
          "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
          "digest": "sha256:0616904aef6b7a3d5b752a186ea902bf16227f8f4bbe01afb440c56f6d845a7f",
          "size": 319
        }
      ]
    },
    "request": {
      "id": "bc908d52-6edf-4970-8f24-9af7106c6e1c",
      "addr": "10.123.123.123",
      "host": "my-custom-registry.internal",
      "method": "PUT",
      "useragent": "containers/5.26.1 (github.com/containers/image)"
    },
    "actor": {
      "name": "admin"
    },
    "source": {
      "addr": "registry-depl-784f44748-6clvq:5000",
      "instanceID": "5d1c2e0d-3b3b-4bba-ab6c-8dc1be1dc058"
    }
  }]
}
```

From which: I want from each event the action, and target repository. Am I supposed to `#permit` the shape and then `#require` the mandatory fields? Like:

```ruby
params.permit(events: [:action, target: [ :repository ] ]).tap do |permitted_params|
  events = permitted_params.require(:events)
  raise ActionController::ParameterMissing.new('STAHP') unless events.is_a?(Array)
  events.each do |event_params|
    event_params.require(:action)
    event_params.require(:target).require(:repository)
  end
end
```

Or am I supposed to just `#require` what I need?:

```ruby
params.require(:events).tap do |events|
  raise ActionController::ParameterMissing.new('STAHP') unless events.is_a?(Array)

  events.each do |event_params|
    events.require(:action)
    event_params.require(:target).require(:repository)
  end
end
```

The [tutorials](https://guides.rubyonrails.org/action_controller_overview.html) or
[API Docs](https://api.rubyonrails.org/classes/ActionController/Parameters.html) don't fully ellaborate what to do with arrays of hashes and that on me raises questions:

- Why `#require` behaves like an array-capable `#fetch` and not like `#permit`???
- Why `#require` doesn't have a dummy type validation? (only array, hash, string is *more* than enough).
- Why `#require` is "dangerous" to fetch terminal values??

If this reaches anybody that can respond this, please reach me on ~~Twitter~~ X @ErCuatrista with an answer.
