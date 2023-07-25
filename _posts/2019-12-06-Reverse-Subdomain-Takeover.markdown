---
layout: post
title:  "A Novel Approach to Subdomain Takeover"
date:   2019-12-06
categories: bugbounty
---

*Subdomain takeover and DNS hijacking have been covered at length by Franz Rosen, Patrik Hudak, and plenty of other people. Rather than rehashing those traditional techniques, this post will explore a novel approach to finding dangling CNAME records. *

### BACKGROUND

Rather than digging through a target organizations assets looking for cnames pointing at lapsed services, we’re going to find services that allow companies to point cnames at them and work backwards. First, a quick explanation for how these services work. 

Imagine you operate a website that we’ll call `legitimatewebsite.com` and you use a third party platform called `jobthing.io` to manage your Careers page. By default, when someone clicks the careers link on your site it forwards them to `legitimatewebsite.jobthing.io`.

Not ideal. You’d much rather keep the user on your domain with something like `careers.legitimatewebsite.com`. Luckily, many third party services offer a Custom/Branded Domain feature using CNAME records. 

In the most basic terms, a CNAME is a DNS record that maps one hostname to another. In the above example, you could create a CNAME record that maps `careers.legitimatewebsite.com` to `legitimatewebsite.jobthing.io`. If jobthing.io is configured to allow this kind of mapping, visiting `careers.legitimatewebsite.com` will load the page normally. 

In the above example we can point the CNAME directly at our jobthing careers page because it lives at the root of a subdomain. If instead, it lived at some file path like `jobthing.io/legitimatewebsite` this approach wouldn’t work because a CNAME can only point to a domain. This is the case for a large number of these third party services and it’s the very thing that enables the technique we’re about to explore. 

If the service hosts each users content at some file path but still wants to offer them a custom domain option, they will provide users with a universal hostname to point CNAMEs at. Every user that wants to use a custom domain will point a CNAME at the same host. The software running on that host will then direct requests to the proper resource according to the hostname they’re using. In the above example, jobthing might have a host called custom.jobthing.io. We would point `careers.legitimatewebsite.com` at that host and jobthing would serve our careers page to anyone reaching that server using that address. 

### TECHNIQUE

In order for a service to be vulnerable to this attack it will need to have a catchall domain, like `custom.jobthing.io`, for customers to point CNAMEs at. Zendesk is an example of a service that allows you to use a custom domain, but doesn’t meet our criteria. SurveyGizmo is an example of one that does meet our criteria. Here’s what I mean: 

Zendesk:

![zendesk]({{ site.url }}/assets/zendesk.png)

SurveyGizmo:

![surveygizmo]({{ site.url }}/assets/sgizmo.png)

Both of these services allow you to point a CNAME at them, but zendesk will have you point it at your unique zendesk url, like `legitimatewebsite.zendesk.com`. SurveyGizmo will have you point your CNAME at a catchall domain called `privatedomain.sgizmo.com`. The key here is that all SurveyGizmo customers are pointing their CNAMEs at that one address. This is what we’ll exploit.

Securitytrails has an enormous collection of DNS data with an API that you can use to find CNAMEs mapped to a given host. We can use this data to find all the CNAMEs pointing at `privatedomain.sgizmo.com` 

We can search through the entries manually on the website and try to find domains that are in scope for some bug bounty, but Securitytrails also offers an API that we can use. The following bash one-liner gives us all 999 CNAMEs pointing to sgizmo in the Securitytrails database. 

```
for i in {1..10}; do curl -s --request POST --url “https://api.securitytrails.com/v1/search/list?page=$i” --header 'apikey: API-KEY-GOES-HERE' --header 'content-type: application/json' --data '{"filter": {"cname": "privatedomain.sgizmo.com"}}' | jq '.records[].hostname' | sed 's/\"//g'; sleep 1; done | tee targets.txt
```

Each result page contains 100 records and there are 999 records in total, that’s why this loops 10 times. Of course, this technique isn’t bound to SurveyGizmo. We can use any service that lets its users point a cname at a catchall domain. 

We can find more with google using dorks like:

`“Custom domain” +cname`

or 

`“Branded domain” +cname`

Once you’ve found a service that uses a catchall domain, look it up on the securitytrails website. If there’s a long enough list of results, use the API to get the full collection. Make sure to run the loop enough times to get all pages. 

Now that we have a complete list of targets, we need to find the ones that have lapsed. There’s a bit of an art to this bit. Look through some of the results manually and see if you can find an example of a lapsed account. If not, see if you can find a pattern that appears on all good results, but that also probably won’t appear on lapsed results; this can be tricky. Next we can use ffuf to find them all. 

If you have a lapsed example, use -mr with a regex pattern that matches something on the page. 

```
Ffuf -w targets -u https://FUZZ -mr ‘not found’
```

If you only have examples of working accounts, you’ll want to use -fr to filter out pages that match the pattern you’ve selected. Again, this is going to be tricky so I wish you luck. It’s not impossible, but you’re going to have to get creative. You need a pattern that is present on all non-lapsed pages, but it also has to be absent on lapsed pages. 

```
Ffuf -w targets.txt -u https://FUZZ -fr ‘title>Welcome’
```

Once you’ve found a lapsed account, google the organization that owns the domain to see if it has a bug bounty program. If it does, the last step is to register an account, hijack the domain, and see what you can do with the service. 

In most cases, you won’t have complete control over what’s hosted on the subdomain, but you may be able to execute script. You’ll want to explore the service to see what you can do with it and then use your imagination to work out how that can translate into a security impact against the target organization. If it’s a service for surveys or job applications or bug tracking, you can request sensitive information from users and abuse the trust relationship between the company’s domain and its customers. If it’s a photo album hosting service you could potentially harm the brand by uploading controversial images. If you can host script then it’s probably stored xss. Get creative!

**If you get any mileage out of this technique I’d love to hear about it. Feel free to send me a message on twitter @healthyoutlet**
