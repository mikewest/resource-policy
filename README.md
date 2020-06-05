# Resource Policy

_[@mikewest](https://github.com/mikewest) & [@kinu](https://github.com/kinu), May 2020_

## A Problem

Servers have a good deal of control over whether or not they deliver a resource in response to a specific request. Fetch Metadata headers provide enough information to evaluate the risk of replying to a given request, and allow servers to implement reasonable heuristics around the way a request was made, and the context in which it will be used. These heuristics can be arbitrarily complex, and [can be a powerful mitigation tool](https://webappsec.dev/assets/pub/Google_IO-Securing_Web_Apps_with_Modern_Platform_Features.pdf#page=48).

Once a resource is delivered, however, the server loses control. Consider a Service Worker which fetches a resource: servers can examine the incoming `Sec-Fetch-*` headers, and make a decision about whether or not to respond to that `fetch()`. Once the resource is cached, however, the server no longer has the ability to prevent the Service Worker laundering it into unexpected contexts. Likewise, servers have little control over the way that content delivered in containers like [Web Bundles](https://web.dev/web-bundles/) will be used.

In both these cases, the decision is binary: deliver the resource to be used in arbitrary ways, or don't. It would be nice to have more granular control on the client side.

## A Proposal

`Cross-Origin-Resource-Policy` is a declarative version of a narrow slice of the server-side logic discussed above. Rather than evaluating the `Sec-Fetch-Site` header themselves, servers can instruct clients to discard a given response if it didn't come from an expected source. Perhaps we could extend servers' capability to make such declarations to include a broader swath of information available on the client-side.

For instance, servers may wish to ensure that a given resource is only loaded into certain contexts: it may be a document intended for framing, in which case it shouldn't be loadable as a script or image or etc. It may be a user-provided image, in which case it shouldn't be loaded as a plugin. Perhaps servers could declare a set of [destinations](https://www.w3.org/TR/fetch-metadata/#sec-fetch-dest-header), [modes](https://www.w3.org/TR/fetch-metadata/#sec-fetch-mode-header), or [site relationships](https://www.w3.org/TR/fetch-metadata/#sec-fetch-site-header) for which a resource is valid, and ask the client to reject it if used elsewhere?

Perhaps we could deprecate `Cross-Origin-Resource-Policy` in favor of a new response header (just `Resource-Policy`?) that has a more granular syntax. Something like the following might be a reasonable syntax to lock a resource to an embedded context:

```http
Resource-Policy: dest=(iframe frame)
```

Or to ensure that a resource is only used as an image:

```http
Resource-Policy: dest=image
```

The header would allow multiple restrictions, and enforce each of them. That is, to guarantee that a resource is used only by same-site endpoints as a script that was requested in CORS mode:

```http
Resource-Policy: site=same-site, dest=script, mode=cors
```

Or (if [w3c/webappsec-fetch-metadata#56](https://github.com/w3c/webappsec-fetch-metadata/issues/56) becomes a thing) to assert that a resource is only used by same-site endpoints as an iframe whose ancestors are also same-site:

```http
Resource-Policy: site=same-site, frame-ancestors=same-site, dest=iframe
```

And so on, and so on...

These headers would be cached along with the response, and could be enforced by the client, even if the cached response was injected by a Service Worker, or extracted from a bundle. 
