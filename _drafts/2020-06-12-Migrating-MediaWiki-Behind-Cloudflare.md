---
title: "Migrating MediaWiki to CloudFlare"
tags: [unity3d]
---
<script type="text/javascript" src="/js/viz.js"></script>
<script type="text/javascript" src="/js/full.render.js"></script>
<script type="text/javascript">
function renderGraph(original, graph) {
  let viz = new Viz();
  viz.renderSVGElement(graph).then(function(element) {
    if (!original.className.includes("center-text")) {
      original.className += "center-text";
    }
    original.appendChild(element);
  }).catch(function(err) {
    console.error(err)
  })
}
</script>
Around one week ago, on June 20th, 2020, the de facto community wiki for Touhou
fans, [touhouwiki.net](https://touhouwiki.net) came under unusually heavy load.
Page response times stretched into 10+ seconds across all subdomains. This is a
public document of the techniques employed by the Touhou wiki sysadmins to
mitigate these problems.

## Background
Touhou wiki is a fansite dedicated to documenting everything about the bullet
hell game series and media franchise [Touhou
Project](https://en.wikipedia.org/wiki/Touhou_Project). First launched as an
independent wiki due to misadministration of the Wikia (now Fandom) back in 2005.
In the 15 years since, the wiki has grown to over 30,000 articles in 10
languages with the service seeing over 1 million pageviews per day from over
100,000 unique visitors.

### Technical Architecture
At the time of the attack, Touhou Wiki has been hosted entirely on a single
standard Linode VPS with 8 CPUs, 32 GB of RAM, and 500 GB. This cost roughly
$240/mo to host the service. The services are split as follows:

<div id="original_arch"></div>
<script type="text/javascript">
renderGraph(document.getElementById('original_arch'), `
digraph OriginalArch {
  rankdir="LR";
  {
    a [label="Outside Internet"]
    b [label="nginx" shape="box"]
    c [label="MediaWiki" shape="box"]
    d [label="MariaDB" shape="cylinder"]
    e [label="memcached" shape="cylinder"]
  }
  a -> b -> c -> d
  d -> c -> b -> a
  c -> e
  e -> c
}`)
</script>

* The front facing nginx server acts as SSL/TLS termination and as a static file
server for bandwidth heavy assets like images. When the attack started, nginx was
not
* The MediaWiki instance sits directly behind the nginx webserver with `php-fpm`.
  `fastcgi_cache` was disabled to due to Touhou Wiki being a [Wiki
  Family](https://www.mediawiki.org/wiki/Manual:Wiki_family).
* The MediaWiki instance stores it's authoritative data in MediaWiki and caches
  objects in the memcached instance.

This is an unusual stack for MediaWiki, as most wikis utilize Apache as a
front-facing webserver. Touhou Wiki migrated to using Nginx years ago due to
difficulty configuring and scaling Apache.

## The Attack
TODO(james7132): Fill this out

#### Attempt 1: Nginx Proxy Caching
<div id="proxy_arch"></div>
<script type="text/javascript">
renderGraph(document.getElementById('proxy_arch'), `
digraph OriginalArch {
  rankdir="LR";
  {
    a [label="Outside Internet"]
    f [label="nginx Proxy Cache" shape="box"]
    b [label="nginx" shape="box"]
    c [label="MediaWiki" shape="box"]
    d [label="MariaDB" shape="cylinder"]
    e [label="memcached" shape="cylinder"]
  }
  a -> f -> b -> c -> d
  d -> c -> b -> f -> a
  c -> e
  e -> c
}`)
</script>

Our first attempt at stemming the problem was to set up an initial proxy cache in
front of the existing nginx server. Fortunately nginx supports binding to
multiple ports, so we just moved the existing binding to port `443` to port
`8000`, and set up a [nginx proxy
cache](https://www.nginx.com/blog/nginx-caching-guide/) with the following
configuration.

```nginx
http {
  ...
  proxy_cache_path /var/cache/nginx keys_zone=thwiki:10m inactive=24h max_size=10g;
  ...
  server {
    listen 443 ssl http2;
    listen [::] 443 ssl http2;
    ...
    location / {
      proxy_pass http://127.0.0.1:8000;
      proxy_cache thwiki;
      proxy_set_header Host $host;
      proxy_buffering on;
      proxy_cache_valid 200  1d;
      proxy_cache_use_stale error timeout invalid_header updating http_500 http_502 http_503 http_504;
    }
  }
}
```

This normally works well as nginx respects upstream server's `Cache-Control`
headers to avoid caching results from logged in users.

We tried this configuration for a day. However, the cache hit ratio was still too
low, only 300,000 of the 3 million requests over the course of the day were hits,
far too low for normal human activity, and a red flag for abnormal automation.

#### Attempt 2: CloudFlare DDoS Protection
The second option we tried was to move the DNS nameservers to CloudFlare to
use the service's anti-DDoS protection.

#### Attempt 3:

#### Attempt 4: CloudFlare WAF, GeoIP Based Filtering

#### Final Results

#### More Optimizations

## Future Work
### CloudFlare Workers

## Conclusions

## Support Us
