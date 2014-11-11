---
layout: post
category: blog
tags: [blog, github]
date: 2014-11-10 21:00:00 -0500
---
{% include JB/setup %}

My blog is using a [custom domain with GitHub Pages](https://help.github.com/articles/setting-up-a-custom-domain-with-github-pages/). As it turns out, this configuration can cause some issues, for example, when the site is indexed by _a_ search engine.

A few people have already posted about this, but [here](http://instantclick.io/github-pages-and-apex-domains) an update on the top of the post says "This is no longer true since at least August 2014. GitHub fixed this!". This note might refer to the bigger issue about the 5 second delay, but it made me think that GitHub does not do redirects anymore. It does:

{% highlight bash %}
$ curl -I rovrov.com
HTTP/1.1 302 Found
Connection: close
Pragma: no-cache
cache-control: no-cache
Location: /
{% endhighlight %}

Note: this does not happen every time, but after a certain timeout the first request oftens gets redirected.

<!-- more -->

Basically, GitHub Pages server replies with a `302` temporary redirect pointing to the same location as requested. The browser will have to request the same exact url once again and hope not to see a `302` this time. I asked GH support if there is something can I do about this, but got this not very encouraging response:

>  you will sometimes receive a 302 Found response in place of a 200 OK response from GitHub Pages sites because of DDoS mitigation software we have in place for all Pages sites
>
> <footer><cite>GitHub support</cite>, November 10, 2014</footer>

Why is this bad for users? Apart from the delay due to the additional round-trip(s) to the server, this may cause issues with searching engines. I would expect crawlers to be smart enough to handle this kind of redirects, but as for some reason my blog was not being properly indexed by google, I submitted a link to [Fetch as Google](https://support.google.com/webmasters/answer/6066467?hl=en). Well, guess what:

<img src="/files/2014-11-09-github-pages-302-redirect/fetch2.png" class="imgmb" alt="Fetch as Google"/>

While it obviously did not follow the redirect, it displays as &#10004;**Complete** in the list and offers to _Submit to Index_ without any warning or whatsoever:

<img src="/files/2014-11-09-github-pages-302-redirect/fetch1.png" class="imgmb" alt="Fetch as Google"/>

Right, submitting empty content to the index sounds like the way to go. I had to fetch some of my urls twice and verify that they got a proper response before submitting. I hope crawlers are a bit smarter in this regard, but you never know _what they might have on their mind_.

According to some posts on this topic, a workaround is to configure a sub-domain such as `www.` using a CNAME pointing to `user.github.io`. I guess I will stick with this if it works better, even though I am not a huge fun of having the `www` prefix around _* Sigh *_