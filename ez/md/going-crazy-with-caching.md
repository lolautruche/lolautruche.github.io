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



# Varnish


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



# User Context

(or how to cache for authenticated users)


## Once upon a time...
* Feature existing in eZ Publish since version 3: "View cache"
* Not using HTTP spec, but file based application cache
* Generated against user roles, policies and limitations


## Some HTTP please?
Has long been considered **impossible** :-(

> They did not know it was impossible,<br>so they did it.<br> <!-- .element: class="fragment" -->
> <br> Mark Twain


## Introducing the User context hash
* Transparent (reverse proxy does the job)
* Computed for every user (footprint)
* Generation can be extended
  * eZ adds roles, policies and limitations
  * Add your custom rules
* Cached by session ID, using HTTP cache of course!


![http_user_context_1](assets/http_user_context_1.png)


![http_user_context_1](assets/http_user_context_2.png)


![http_user_context_1](assets/http_user_context_3.png)


## ...And they lived happily ever after
* Feature available in [FOSHttpCache library](http://foshttpcache.readthedocs.org/en/latest/)
* Integrates seamlessly with Symfony with [FOSHttpCacheBundle](http://foshttpcachebundle.readthedocs.org/en/latest/)
* Works with Varnish 3 & 4
* ...And Symfony reverse proxy! <!-- .element: class="fragment" -->



# Let's play!

![VCL](assets/varnish-stage.png)


# Use-case: Content subscription


## Anonymous users may only see an abstract
![article](assets/article_anonymous.png)


## Granted users can read the full article
![article](assets/article_subscriber.png)



## Cleaning cookies

```
sub vcl_recv {
  set req.http.cookie = ";" + req.http.cookie;
  set req.http.cookie = regsuball(req.http.cookie, "; +", ";");
  set req.http.cookie = regsuball(req.http.cookie, ";(PHPSESSID)=","; \1=");
  set req.http.cookie = regsuball(req.http.cookie, ";[^ ][^;]*", "");
  set req.http.cookie = regsuball(req.http.cookie, "^[; ]+|[; ]+$", "");
}
```


TODO
Caching the context lookup vs doing a paywall - maybe also mention mod-sendfile while we are at it...


## Combining with Vary header

* Different caching rules
    * No caching
    * Vary: user context
    * Vary: Cookie
* ESI for different rules for specific fragments


## Edge Side Includes

*Like server side include, but on Varnish*

* Content embeds URLs to parts of the content
* Varnish fetches and caches elements separatly
* Individual caching rules per fragment
* E.g. only some elements vary on cookie, different TTL, ...
* Symfony has built-in support


TODO FOSHttpCacheBundle header configuration examples...



# Thanks!

## @dbu

## @jvieilledent
