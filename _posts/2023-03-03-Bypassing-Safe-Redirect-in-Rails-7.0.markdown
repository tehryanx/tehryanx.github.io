---
layout: post
title:  "Bypassing Safe-Redirect in Rails 7.0"
date:   2023-03-03
categories: bugbounty
---

*Yet another parsing differential bug*

# Safe-Redirect

The normal pattern for throwing a 302 redirect in a rails application is by using the built-in method `redirect_to`. As of rails v7.0, the default behaviour is to only allow relative redirects to locations on the same origin domain, unless the `allow_other_host` flag is set to true. This functionality was designed to mitigate open redirect vulnerabilities, but a bug in the implementation allowed for a bypass. Let's discuss!

---

First, let's look at how this functionality is intended to work. I've set up a simple rails app where the `/redirect` route will accept a GET param called `to` and call `redirect_to` using that value as the path to redirect the user to. The controller looks like this:

```
class RedirectController < ApplicationController
  def index
    unless params[:to].nil?
      redirect_to(params[:to])
    end
  end
end
```

When `params[:to]` is a relative path this works as you'd expect. For example, `https://localhost:3000/redirect?to=/home` throws a 302 that redirects us to `/home`:

![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/b8558cd1-58d0-4848-afb1-86fda288d159)

If we insteaed give it a URL pointing to a different domain it throws an UnsafeRedirectError:

![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/28c23734-578b-4b23-b7c5-669663f2b58a)

The goal is to bypass this control and get an offsite redirect to fire without raising this exception. This ended up being possible because ruby parses URIs differently than modern browsers do. Let's start by digging into rails' safe redirect code. 

---

When calling `redirect_to` without enabling the `allow_other_host` flag, rails passes the url off to a validation function called `_url_host_allowed?`. 
```
def _url_host_allowed?(url)
  host = URI(url.to_s).host
  host == request.host || host.nil? && url.to_s.start_with?("/")
rescue ArgumentError, URI::Error
  false
end
```
This function attempts to validate that the path a user is being redirected to is on the same host as the one they're being redirected from. In order to satisfy this validation one of two conditions must be met. These are:

- `host == request.host` - which passes if the host in the redirect uri matches the applications host.
- `host.nil? && url.to_s.start_with?("/")` - which is supposed to match only on relative uris like `/somepath` but not like `http://evil.com`

Only one of these conditions must to be satisfied to pass the validation, and it turns out that the second one has a flaw. We can trick this validator into believing that an absolute URI pointing to some external host is actually a relative URI pointing to some path on the current host. Based on the above code, rails is defining a relative uri as one that

1. has a `nil` host
2. starts with a `/`

In order to make this work we will chain together a convenience built into modern browsers and a quirk of ruby's URI parser.

First, URIs that start with a `/` are normally relative paths pointing to something on the current host, while uris starting with a scheme like `https://` are absolute uris that can point to any host on the internet. As a convenience, modern browsers will take uris starting with `//` and convert that to the full scheme format `https://`. You can test this right now by typing the following into the javascript console in your browser: `location.href="//google.com"`.

This is useful for us because it means that it's possible to construct an absolute uri that would pass `uri.to_s.start_with?("/")`.

Second, Ruby's URI parser conforms to a different spec than modern web browsers, and a subtle difference between these two specifications will allow us to bypass the `host.nil?` requirement. Modern browsers obey the whatwg spec, which requires the parser to fix urls with [too many slashes after the scheme](https://url.spec.whatwg.org/#example-url-parsing):

![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/ffecaa87-c82d-49ff-9da1-7b39fff519f3)

Ruby's URI parser sees this differently. If supplied a URI with 3 slashes in the scheme, it will parse this as though the host is simply the empty string between the second and third slash. Same is true when the scheme is omitted:
```
irb(main):001:0> URI("https:///test.com/").host
=> nil
irb(main):002:0> URI("///test.com/").host
=> nil
```
NOTE: I reported this behaviour to Ruby last year, but they determined this to not be a bug as this is technically spec compliant, just a different set of specs than browsers use. 

This means that a url like `///evil.com` given to ruby will return true on `host.nil?`, but when given to a browser will be seen as pointing to `https://evil.com`

![image](https://github.com/tehryanx/tehryanx.github.io/assets/8878295/7b7ffde7-8b21-4f8f-ba5d-060093cb856b)

---

This was a universal bypass for Rails 7.0's safe-redirect functionality. It has now been patched. 
