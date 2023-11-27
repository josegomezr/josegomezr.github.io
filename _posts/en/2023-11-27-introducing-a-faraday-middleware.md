---
title: Introducing `Faraday::Midddlewares::BuildService`
layout: post
tags: [blog, ruby, http, rubygem]
categories: [blog, programming]
lang: en
---

It's been a journey since [_Multi-factor authentication on SUSE’s Build
Service_ @ suse.com blog][suse-com-mfa-on-bs], but now there's code to show! 🎉

A [Faraday][faraday] middleware for Build Service Authentication is now live!

Check it out at https://github.com/SUSE/faraday-middleware-bs-auth

<!--more-->

# So what's the need for this?

Cuz... don't repeat code? Yep, that pretty much sums it.

I don't know whether it's well known or technical-enough to be public
information, but there's a special authentication mechanism for the build
service (see [the blogpost][suse-com-mfa-on-bs]) that we needed some workloads
to support.

The majority of the workloads we maintain are written in Ruby, and after
copypasting the original implementation once, we decided to make a gem out of
it.

So, yep, DRY.

# What's special

Honestly? Nothing, it's just a middleware that will retry requests with the
requested authentication.

The [blogpost @ suse.com][suse-com-mfa-on-bs] really gets to the technical
reference and the gem is just an implementation of it.

[suse-com-mfa-on-bs]: https://www.suse.com/c/multi-factor-authentication-on-suses-build-service/
[faraday]: https://github.com/lostisland/faraday
