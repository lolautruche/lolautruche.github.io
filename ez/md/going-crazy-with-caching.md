# Caching content of logged in users

### Going crazy with caching

By [Jérôme Vieilledent](https://github.com/lolautruche) / [@jvieilledent](http://twitter.com/jvieilledent)
and [David Buchmann](http://davidbu.ch/mann/) / [@dbu](http://twitter.com/dbu)

Symfony Live Paris 2015


## HTTP Caching

* Why re-render content that does not change?
* **Scaling** and response time
* Less servers, more users


## Architecture

![reverse proxy](assets/varnish-reverse-proxy.png)


## HTTP cache control

```
Cache-Control: s-maxage=3600, max-age=900
```

1. s-maxage
2. max-age
3. Expires
4. Default to default_ttl if nothing specified


## Do **not** cache

```
Cache-Control: s-maxage=0, private, no-cache
```

* Varnish 3 does not respect no-cache by default, but s-maxage=0.
* Varnish 4 respects both s-maxage=0 and no-cache.


## Keep variants apart

### Request
```
Accept: application/json
```
### Response
```
Vary: Accept
```


## The Varnish state engine
![VCL](assets/varnish-vcl.png)


## VCL Hooks
* vcl_recv: entry point, lookup or pass
* vcl_fetch: receive from backend, cache or not
* vcl_deliver: remove headers not for client
* vcl_hash: determine cache key
* vcl_hit: found in cache, deliver or pass
* vcl_miss: pass or fetch
* vcl_pipe: do not alter request
* vcl_error: define error page


## Default behaviour

* Only ever cache GET / HEAD
* Never cache request with COOKIE / AUTHORIZATION
* Never cache response with SET-COOKIE
* Only response with success status, redirects and 404, 410



## TODO: user context slides



## Edge Side Includes

Like server side include, but on Varnish:

* Content embeds URLs to parts of the content
* Varnish fetches and caches elements separatly
* Individual caching rules per fragment
* E.g. only some elements vary on cookie, different TTL, ...
* Symfony has built-in support

