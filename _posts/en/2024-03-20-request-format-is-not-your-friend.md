---
title: "`request.format` is not your friend"
layout: post
tags: [blog]
categories: [blog, til, rails]
lang: 
---

How many times you wanted to have your endpoint exclusively receive requests in JSON format?

<!--more-->

I bet probably wrote: 

```ruby
head :unprocessable_entity unless request.format.json?
```

and thought: _My job here is done!_. Well not so fast...


What you probably meant is:

> If the request is not `application/json`, then reject it with `422 Unprocessable Entity`.

Like any human being who wants to use HTTP codes appropriately. So you do the rational thing: put it to work

```bash
# I'm exaggerating my curl profficiency just to prove a point

curl -sSL -w "%{http_code}" -o/dev/null localhost:3000/your-controller/path?format=json
=> 200 # expected, params[:format] == :json

curl -sSL -w "%{http_code}" -o/dev/null -H 'Content-Type: application/json' localhost:3000/your-controller/path
=> 422 # WTF?
```

What the actual? Isn't that a JSON??? Well...

It all boils down to [that tricky `request.format` bit][request-format].

```ruby
# actionpack/lib/action_dispatch/http/mime_negotiation.rb
# [...]
def format(_view_path = nil)
  formats.first || Mime::NullType.instance
end

def formats
  fetch_header("action_dispatch.request.formats") do |k|
    v = if params_readable?
      Array(Mime[parameters[:format]]) # Writer's Note: this means query-string or request body has: format=json
    elsif use_accept_header && valid_accept_header # Writer's Note: this means: Has "Accept: application/json" ?
      accepts.dup
    elsif extension_format = format_from_path_extension
      [extension_format] # Writer's Note: this means "request uri ends_with '.json'?"
    elsif xhr?
      [Mime[:js]]
    else
      [Mime[:html]]
    end

    v.select! do |format|
      format.symbol || format.ref == "*/*"
    end

    set_header k, v
  end
end
# [...]
```

TL;DR: what you told rails was:

> If the request does not [either of the following]
>  * have `format=json` in either query string or parseable request body.
>  * have an `Accept` header that is recognized as json: `application/json`, `application/jsonrequest`, `application/problem+json`
>  * have a path ending `.json` extension
>  Reject it.

And that's not _quite_ equivalent... You see, from [MDN Docs on HTTP `Accept` Header][MDN-Accept]:

> The Accept request HTTP header indicates which content types, expressed as
  MIME types, **the client is able to understand**. [...]

The `Accept` header is used strictly from the client perspective to say: _"Please gimme a %content-type%, that's what I can read"_.

<small>_Side note: It's fun to me too that to check the format the body _is_ parsed 😂._</small>

In any case what you probably meant to write was:

```ruby
head :unprocessable_entity unless request.content_mime_type.json?
```

Which really says: _"Reject the request if the client did not provide a json"_

This _"new friend" is defined a [couple of lines above our mischievous pal `ActionDispatch::Request#format`]([request-content-mime])

```ruby
def content_mime_type
  fetch_header("action_dispatch.request.content_type") do |k|
    v = if get_header("CONTENT_TYPE") =~ /^([^,;]*)/
      Mime::Type.lookup($1.strip.downcase)
    else
      nil
    end
    set_header k, v
  rescue ::Mime::Type::InvalidMimeType => e
    raise InvalidType, e.message
  end
end
```

This one instead tries to lookup the value of the `Content-Type` header, which according to [MDN Docs on HTTP `Content-Type`][MDN-Content-Type]:

> The Content-Type representation header is used to indicate the original media type of the resource (prior to any content encoding applied for sending).
> [...]
> In requests, (such as POST or PUT), the client tells the server **what type of data is actually sent**.

**Lesson learned:** `request.format` is for **Response format negotiation** (What to
send to the client), DO NOT ATTEMPT TO USE IT TO DETERMINE IF A CLIENT SENT YOU
A JSON, IT WILL FAIL MISERABLY AND YOU'LL JUST PUT A `.json` AT THE END AND RUIN YOUR SHINY BEAUTIFUL SMART ENDPOINT

[MDN-Accept]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept
[MDN-Content-Type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type
[request-format]: https://github.com/rails/rails/blob/v7.1.3.2/actionpack/lib/action_dispatch/http/mime_negotiation.rb#L75-L85
[request-content-mime]: https://github.com/rails/rails/blob/v7.1.3.2/actionpack/lib/action_dispatch/http/mime_negotiation.rb#L36-L47
