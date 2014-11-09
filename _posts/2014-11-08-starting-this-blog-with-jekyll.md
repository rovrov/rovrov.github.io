---
layout: post
category : blog
tags : [blog, jekyll, bootstrap]
---
{% include JB/setup %}

A quick search for blogging platforms and hostings led me to GitHub Pages offering to host a site directly from a GitHub repo using Jekyll and Markdown. After seeing a few people migrating their blogs from Wordpress to Jekyll and not being too unhappy, I figured I should give it a try. Of course, then I had to write this mandatory _look-now-I-am-a-Jekyll-hacker_ post.

I wish there were a little more structure to this blogging platform. For now what I've seen is people cloning another blogs and then patching all around the code using snippets from the web. Having some sort of components and being able to inlcude them as building blocks would be nicer... well, it is not there yet I guess.

[Jekyll Bootstrap](http://jekyllbootstrap.com/) has some structure, and even though it seemed unfinished and somewhat abandoned, I decided to use it as a starting point and modify to my needs. First thing I did was `rake theme:switch name="bootstrap-3"`.

<!-- more -->

A lot of things come with JB out-of-the-box, such as RSS/Atom, sitemap, tags and categories (not sure if I need both of those though). Enabling Disqus and GA was only a matter of modifying `_config.yml`. GA was not working for me when served locally with `jekyll serve`, so I had to find out that it is only enabled when `site.safe` is set (e.g. `jekyll serve --safe`).

First I thought code highlighting was not working but _I just couldn't see it :)_ The css classes were actually applied to the code, but the css with the corresponding style definitions file `pygments.css` was missing from the distribution for some reason.

Post previews and pagination did not come with JB. [Read More without plugin](http://truongtx.me/2013/05/01/jekyll-read-more-feature-without-any-plugin/) describes an easy way to get post previews. Pagination support comes directly from Jekyll, so I just enabled it in `_config.yaml`

{% highlight yaml %}
paginate: 5
paginate_path: "blog/page:num"  # my blog lives at /blog/
{% endhighlight %}

and used [this pagination snippet](http://jekyllrb.com/docs/pagination/#render-the-paginated-posts) replacing the pangination links part with this bootstrap 3 version:

{% highlight html %}
{% raw %}
<!-- Pagination links -->
<nav>
  <ul class="pager">
    <li class="previous{% unless paginator.previous_page %} disabled{% endunless %}"><a href="{{paginator.previous_page_path | replace: '/index.html', '/'}}">&larr; Older</a></li>
    <li class="next{% unless paginator.next_page %} disabled{% endunless %}"><a href="{{paginator.next_page_path}}">Newer &rarr;</a></li>
  </ul>
</nav>
{% endraw %}
{% endhighlight %}

I also modified a few visual things here and there, probably had to figure out and change more than I originally expected but that was fun.