# Going crazy with <br>HTTP caching
### Caching content for authenticated users

By [Jérôme Vieilledent](https://github.com/lolautruche) / [@jvieilledent](http://twitter.com/jvieilledent)
<br>and [David Buchmann](http://davidbu.ch/mann/) / [@dbu](http://twitter.com/dbu)

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




# FOSHttpCacheBundle


## Features

* Request matcher for response headers
* Annotations for response headers and invalidation
* Active cache invalidation
* Cache tagging and invalidation
* User Context


## Supported

* Varnish
* Nginx
* Symfony HttpCache


## Testing

* Well tested
* Reusable base test for functional cache tests




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



# Use-case: Content subscription


## Anonymous users may only see an abstract
![article](assets/article_anonymous.png)


## Granted users can read the full article
![article](assets/article_subscriber.png)


## Let's play!

![VCL](assets/varnish-stage.png)



## Watch out

* Clean the cookie header when fetching the hash
* Make sure the Vary headers are correct


## Combining with Vary header

* Different caching rules
    * No caching
    * Vary: user context
    * Vary: Cookie
* ESI for different rules for specific fragments



# Thanks!

## @dbu

## @jvieilledent

#### http://lolautruche.github.io/ez/going-crazy-with-caching.html
#### Further slides on [ezsystems.github.io/slides](http://ezsystems.github.io/slides/)
